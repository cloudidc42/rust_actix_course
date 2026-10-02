# Part 057: GraphQL with async-graphql 🔷

## 🎯 เป้าหมายของ Part นี้

- async-graphql crate setup
- Schema definition (Object, InputObject, Enum)
- Query resolvers
- Mutation resolvers
- Subscription resolvers
- Authentication ใน GraphQL context
- DataLoader สำหรับแก้ปัญหา N+1
- GraphQL playground
- สร้าง Blog GraphQL API

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
async-graphql = { version = "7", features = ["chrono", "uuid"] }
async-graphql-actix-web = "7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-native-tls", "chrono", "uuid"] }
async-trait = "0.1"
dataloader = "0.17"
tokio-stream = "0.1"
```

---

## 2. Schema Definition

### 2.1 Object Types

```rust
// src/graphql/types.rs
use async_graphql::*;
use chrono::{DateTime, Utc};
use uuid::Uuid;

// Blog post type
#[derive(Debug, Clone, SimpleObject)]
pub struct Post {
    pub id: Uuid,
    pub title: String,
    pub slug: String,
    pub content: String,
    pub excerpt: Option<String>,
    pub published: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    // ไม่ใส่ author_id โดยตรง - ใช้ resolver แทน
}

// Author type
#[derive(Debug, Clone, SimpleObject)]
pub struct Author {
    pub id: Uuid,
    pub name: String,
    pub email: String,
    pub bio: Option<String>,
    pub avatar_url: Option<String>,
    pub created_at: DateTime<Utc>,
}

// Comment type
#[derive(Debug, Clone, SimpleObject)]
pub struct Comment {
    pub id: Uuid,
    pub content: String,
    pub post_id: Uuid,
    pub author_id: Uuid,
    pub created_at: DateTime<Utc>,
}

// Tag type
#[derive(Debug, Clone, SimpleObject)]
pub struct Tag {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
}

// Post status enum
#[derive(Debug, Clone, Enum, Copy, PartialEq, Eq)]
pub enum PostStatus {
    Draft,
    Published,
    Archived,
}

// Pagination info
#[derive(Debug, Clone, SimpleObject)]
pub struct PageInfo {
    pub total: i64,
    pub page: i32,
    pub per_page: i32,
    pub has_next_page: bool,
}

// Paginated posts
#[derive(Debug, Clone, SimpleObject)]
pub struct PostConnection {
    pub nodes: Vec<Post>,
    pub page_info: PageInfo,
}
```

### 2.2 Input Types

```rust
// src/graphql/inputs.rs
use async_graphql::*;
use uuid::Uuid;

#[derive(Debug, InputObject)]
pub struct CreatePostInput {
    pub title: String,
    pub content: String,
    pub excerpt: Option<String>,
    #[graphql(default = false)]
    pub published: bool,
    pub tag_ids: Option<Vec<Uuid>>,
}

#[derive(Debug, InputObject)]
pub struct UpdatePostInput {
    pub title: Option<String>,
    pub content: Option<String>,
    pub excerpt: Option<String>,
    pub published: Option<bool>,
    pub tag_ids: Option<Vec<Uuid>>,
}

#[derive(Debug, InputObject)]
pub struct PostFilter {
    pub status: Option<super::types::PostStatus>,
    pub author_id: Option<Uuid>,
    pub tag_slug: Option<String>,
    pub search: Option<String>,
}

#[derive(Debug, InputObject)]
pub struct PaginationInput {
    #[graphql(default = 1)]
    pub page: i32,
    #[graphql(default = 20)]
    pub per_page: i32,
}

#[derive(Debug, InputObject)]
pub struct CreateCommentInput {
    pub post_id: Uuid,
    pub content: String,
}

#[derive(Debug, InputObject)]
pub struct RegisterInput {
    pub name: String,
    pub email: String,
    pub password: String,
}

#[derive(Debug, InputObject)]
pub struct LoginInput {
    pub email: String,
    pub password: String,
}
```

---

## 3. GraphQL Context

```rust
// src/graphql/context.rs
use async_graphql::Context;
use sqlx::PgPool;
use std::sync::Arc;

pub struct GraphQLContext {
    pub db: PgPool,
    pub current_user: Option<CurrentUser>,
}

#[derive(Debug, Clone)]
pub struct CurrentUser {
    pub id: uuid::Uuid,
    pub email: String,
    pub role: UserRole,
}

#[derive(Debug, Clone, PartialEq)]
pub enum UserRole {
    Admin,
    Author,
    Reader,
}

pub trait GraphQLContextExt {
    fn get_db(&self) -> &PgPool;
    fn get_current_user(&self) -> Option<&CurrentUser>;
    fn require_auth(&self) -> async_graphql::Result<&CurrentUser>;
    fn require_role(&self, role: UserRole) -> async_graphql::Result<&CurrentUser>;
}

impl GraphQLContextExt for Context<'_> {
    fn get_db(&self) -> &PgPool {
        self.data_unchecked::<PgPool>()
    }

    fn get_current_user(&self) -> Option<&CurrentUser> {
        self.data_opt::<CurrentUser>()
    }

    fn require_auth(&self) -> async_graphql::Result<&CurrentUser> {
        self.get_current_user()
            .ok_or_else(|| async_graphql::Error::new("Authentication required"))
    }

    fn require_role(&self, required_role: UserRole) -> async_graphql::Result<&CurrentUser> {
        let user = self.require_auth()?;
        if user.role == required_role || user.role == UserRole::Admin {
            Ok(user)
        } else {
            Err(async_graphql::Error::new("Insufficient permissions"))
        }
    }
}
```

---

## 4. Query Resolvers

```rust
// src/graphql/query.rs
use async_graphql::*;
use uuid::Uuid;
use crate::graphql::{
    types::*,
    inputs::*,
    context::GraphQLContextExt,
};

pub struct QueryRoot;

#[Object]
impl QueryRoot {
    // Get single post
    async fn post(&self, ctx: &Context<'_>, id: Uuid) -> Result<Option<Post>> {
        let db = ctx.get_db();

        let post = sqlx::query_as!(
            Post,
            r#"
            SELECT id, title, slug, content, excerpt, published, created_at, updated_at
            FROM posts
            WHERE id = $1
            "#,
            id
        )
        .fetch_optional(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(post)
    }

    // Get post by slug
    async fn post_by_slug(
        &self,
        ctx: &Context<'_>,
        slug: String,
    ) -> Result<Option<Post>> {
        let db = ctx.get_db();

        let post = sqlx::query_as!(
            Post,
            "SELECT id, title, slug, content, excerpt, published, created_at, updated_at FROM posts WHERE slug = $1",
            slug
        )
        .fetch_optional(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(post)
    }

    // List posts with pagination and filter
    async fn posts(
        &self,
        ctx: &Context<'_>,
        filter: Option<PostFilter>,
        pagination: Option<PaginationInput>,
    ) -> Result<PostConnection> {
        let db = ctx.get_db();
        let page = pagination.as_ref().map(|p| p.page).unwrap_or(1);
        let per_page = pagination.as_ref().map(|p| p.per_page).unwrap_or(20);
        let offset = (page - 1) * per_page;

        // ดึง total count
        let total: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM posts WHERE published = true"
        )
        .fetch_one(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?
        .unwrap_or(0);

        // ดึง posts
        let posts = sqlx::query_as!(
            Post,
            r#"
            SELECT id, title, slug, content, excerpt, published, created_at, updated_at
            FROM posts
            WHERE published = true
            ORDER BY created_at DESC
            LIMIT $1 OFFSET $2
            "#,
            per_page as i64,
            offset as i64
        )
        .fetch_all(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(PostConnection {
            nodes: posts,
            page_info: PageInfo {
                total,
                page,
                per_page,
                has_next_page: (offset as i64 + per_page as i64) < total,
            },
        })
    }

    // Get author
    async fn author(&self, ctx: &Context<'_>, id: Uuid) -> Result<Option<Author>> {
        let db = ctx.get_db();

        let author = sqlx::query_as!(
            Author,
            "SELECT id, name, email, bio, avatar_url, created_at FROM users WHERE id = $1",
            id
        )
        .fetch_optional(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(author)
    }

    // Get current user
    async fn me(&self, ctx: &Context<'_>) -> Result<Author> {
        let user = ctx.require_auth()?;
        let db = ctx.get_db();

        sqlx::query_as!(
            Author,
            "SELECT id, name, email, bio, avatar_url, created_at FROM users WHERE id = $1",
            user.id
        )
        .fetch_one(db)
        .await
        .map_err(|e| Error::new(e.to_string()))
    }
}
```

---

## 5. Mutation Resolvers

```rust
// src/graphql/mutation.rs
use async_graphql::*;
use uuid::Uuid;
use crate::graphql::{types::*, inputs::*, context::GraphQLContextExt};

pub struct MutationRoot;

#[Object]
impl MutationRoot {
    // Create post
    async fn create_post(
        &self,
        ctx: &Context<'_>,
        input: CreatePostInput,
    ) -> Result<Post> {
        let user = ctx.require_auth()?;
        let db = ctx.get_db();

        let slug = slugify(&input.title);

        let post = sqlx::query_as!(
            Post,
            r#"
            INSERT INTO posts (id, title, slug, content, excerpt, published, author_id, created_at, updated_at)
            VALUES ($1, $2, $3, $4, $5, $6, $7, NOW(), NOW())
            RETURNING id, title, slug, content, excerpt, published, created_at, updated_at
            "#,
            Uuid::new_v4(),
            input.title,
            slug,
            input.content,
            input.excerpt,
            input.published,
            user.id,
        )
        .fetch_one(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(post)
    }

    // Update post
    async fn update_post(
        &self,
        ctx: &Context<'_>,
        id: Uuid,
        input: UpdatePostInput,
    ) -> Result<Post> {
        let user = ctx.require_auth()?;
        let db = ctx.get_db();

        // ตรวจสอบว่าเป็น post ของ user นี้
        let existing = sqlx::query!(
            "SELECT author_id FROM posts WHERE id = $1",
            id
        )
        .fetch_optional(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?
        .ok_or_else(|| Error::new("Post not found"))?;

        if existing.author_id != user.id {
            return Err(Error::new("You can only edit your own posts"));
        }

        let post = sqlx::query_as!(
            Post,
            r#"
            UPDATE posts
            SET
                title = COALESCE($1, title),
                content = COALESCE($2, content),
                excerpt = COALESCE($3, excerpt),
                published = COALESCE($4, published),
                updated_at = NOW()
            WHERE id = $5
            RETURNING id, title, slug, content, excerpt, published, created_at, updated_at
            "#,
            input.title,
            input.content,
            input.excerpt,
            input.published,
            id,
        )
        .fetch_one(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(post)
    }

    // Delete post
    async fn delete_post(&self, ctx: &Context<'_>, id: Uuid) -> Result<bool> {
        let user = ctx.require_auth()?;
        let db = ctx.get_db();

        let result = sqlx::query!(
            "DELETE FROM posts WHERE id = $1 AND author_id = $2",
            id,
            user.id
        )
        .execute(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(result.rows_affected() > 0)
    }

    // Add comment
    async fn add_comment(
        &self,
        ctx: &Context<'_>,
        input: CreateCommentInput,
    ) -> Result<Comment> {
        let user = ctx.require_auth()?;
        let db = ctx.get_db();

        let comment = sqlx::query_as!(
            Comment,
            r#"
            INSERT INTO comments (id, content, post_id, author_id, created_at)
            VALUES ($1, $2, $3, $4, NOW())
            RETURNING id, content, post_id, author_id, created_at
            "#,
            Uuid::new_v4(),
            input.content,
            input.post_id,
            user.id,
        )
        .fetch_one(db)
        .await
        .map_err(|e| Error::new(e.to_string()))?;

        Ok(comment)
    }
}

fn slugify(s: &str) -> String {
    s.to_lowercase()
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

## 6. Subscription Resolvers

```rust
// src/graphql/subscription.rs
use async_graphql::*;
use tokio_stream::Stream;
use crate::graphql::types::*;

pub struct SubscriptionRoot;

#[Subscription]
impl SubscriptionRoot {
    // Subscribe to new posts
    async fn new_posts(
        &self,
        ctx: &Context<'_>,
    ) -> impl Stream<Item = Post> {
        let broadcaster = ctx.data_unchecked::<PostBroadcaster>();
        broadcaster.subscribe()
    }

    // Subscribe to new comments on a post
    async fn post_comments(
        &self,
        ctx: &Context<'_>,
        post_id: uuid::Uuid,
    ) -> impl Stream<Item = Comment> {
        let broadcaster = ctx.data_unchecked::<CommentBroadcaster>();
        broadcaster.subscribe_to_post(post_id)
    }
}

// Broadcaster helper
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;
use tokio_stream::StreamExt;

pub struct PostBroadcaster {
    sender: broadcast::Sender<Post>,
}

impl PostBroadcaster {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(100);
        Self { sender }
    }

    pub fn broadcast(&self, post: Post) {
        let _ = self.sender.send(post);
    }

    pub fn subscribe(&self) -> impl Stream<Item = Post> {
        BroadcastStream::new(self.sender.subscribe())
            .filter_map(|r| r.ok())
    }
}

pub struct CommentBroadcaster {
    sender: broadcast::Sender<Comment>,
}

impl CommentBroadcaster {
    pub fn new() -> Self {
        let (sender, _) = broadcast::channel(100);
        Self { sender }
    }

    pub fn broadcast(&self, comment: Comment) {
        let _ = self.sender.send(comment);
    }

    pub fn subscribe_to_post(
        &self,
        post_id: uuid::Uuid,
    ) -> impl Stream<Item = Comment> {
        BroadcastStream::new(self.sender.subscribe())
            .filter_map(|r| r.ok())
            .filter(move |c| c.post_id == post_id)
    }
}
```

---

## 7. DataLoader (N+1 Solution)

```rust
// src/graphql/dataloader.rs
use async_graphql::dataloader::{DataLoader, Loader};
use async_trait::async_trait;
use sqlx::PgPool;
use std::collections::HashMap;
use uuid::Uuid;
use crate::graphql::types::Author;

pub struct AuthorLoader {
    db: PgPool,
}

impl AuthorLoader {
    pub fn new(db: PgPool) -> Self {
        Self { db }
    }
}

#[async_trait]
impl Loader<Uuid> for AuthorLoader {
    type Value = Author;
    type Error = async_graphql::Error;

    async fn load(
        &self,
        keys: &[Uuid],
    ) -> Result<HashMap<Uuid, Self::Value>, Self::Error> {
        // Load ทุก authors ในครั้งเดียว (batch query)
        let authors = sqlx::query_as!(
            Author,
            r#"
            SELECT id, name, email, bio, avatar_url, created_at
            FROM users
            WHERE id = ANY($1)
            "#,
            keys as &[Uuid],
        )
        .fetch_all(&self.db)
        .await
        .map_err(|e| async_graphql::Error::new(e.to_string()))?;

        Ok(authors.into_iter().map(|a| (a.id, a)).collect())
    }
}

// Usage ใน Post resolver
use async_graphql::{ComplexObject, Context, Result};

// ต้องใช้ #[derive(SimpleObject)] พร้อม #[graphql(complex)]
#[ComplexObject]
impl Post {
    async fn author(&self, ctx: &Context<'_>) -> Result<Option<Author>> {
        let loader = ctx.data_unchecked::<DataLoader<AuthorLoader>>();
        let author_id: Uuid = ctx.data_unchecked::<PostAuthorIds>()
            .get(&self.id)
            .copied()
            .unwrap_or_default();

        loader.load_one(author_id).await
    }

    async fn comments(&self, ctx: &Context<'_>) -> Result<Vec<Comment>> {
        let db = ctx.data_unchecked::<PgPool>();

        sqlx::query_as!(
            Comment,
            "SELECT id, content, post_id, author_id, created_at FROM comments WHERE post_id = $1 ORDER BY created_at ASC",
            self.id
        )
        .fetch_all(db)
        .await
        .map_err(|e| async_graphql::Error::new(e.to_string()))
    }
}

pub struct PostAuthorIds(HashMap<Uuid, Uuid>); // post_id -> author_id
```

---

## 8. Actix-web Integration

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, HttpRequest, HttpResponse};
use async_graphql::{EmptySubscription, Schema};
use async_graphql_actix_web::{GraphQLRequest, GraphQLResponse, GraphQLSubscription};
use sqlx::PgPool;

type BlogSchema = Schema<
    crate::graphql::query::QueryRoot,
    crate::graphql::mutation::MutationRoot,
    EmptySubscription,
>;

async fn graphql_handler(
    schema: web::Data<BlogSchema>,
    req: HttpRequest,
    gql_req: GraphQLRequest,
) -> GraphQLResponse {
    // Extract auth token
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "));

    let mut request = gql_req.into_inner();

    // เพิ่ม current user เข้า context
    if let Some(token) = token {
        // validate token และ get user...
        // request = request.data(current_user);
    }

    schema.execute(request).await.into()
}

async fn graphql_playground() -> HttpResponse {
    let html = async_graphql::http::playground_source(
        async_graphql::http::GraphQLPlaygroundConfig::new("/graphql")
            .subscription_endpoint("/graphql/ws"),
    );

    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(html)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let db = PgPool::connect("postgres://localhost/blog_db")
        .await
        .expect("Failed to connect to DB");

    let schema = Schema::build(
        crate::graphql::query::QueryRoot,
        crate::graphql::mutation::MutationRoot,
        EmptySubscription,
    )
    .data(db.clone())
    .finish();

    let schema_data = web::Data::new(schema);

    HttpServer::new(move || {
        App::new()
            .app_data(schema_data.clone())
            .route("/graphql", web::post().to(graphql_handler))
            .route("/graphql", web::get().to(graphql_playground))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

✅ async-graphql setup  
✅ Object, InputObject, Enum types  
✅ Query resolvers  
✅ Mutation resolvers  
✅ Subscription resolvers  
✅ Authentication ใน context  
✅ DataLoader สำหรับ N+1 problem  
✅ GraphQL playground  

### Exercise

1. เพิ่ม file upload ผ่าน GraphQL
2. สร้าง custom scalar types
3. เพิ่ม field-level authorization
4. Implement cursor-based pagination

---

*[← Part 056: Payment Gateway Integration](../part_056/README.md) | [Part 058: gRPC with Tonic →](../part_058/README.md)*
