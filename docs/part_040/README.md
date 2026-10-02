# Part 040: Full REST API with Database 🏗️

## 🎯 เป้าหมายของ Part นี้

- สร้าง REST API ที่สมบูรณ์ด้วย Actix-web + SQLx
- Clean architecture: Handler → Service → Repository
- Input validation, Error handling, Pagination
- JWT Authentication integration
- Production-ready project structure

---

## 1. Project Structure

```
rest_api/
├── Cargo.toml
├── .env
├── migrations/
│   ├── 20240101_001_create_categories.sql
│   └── 20240101_002_create_products.sql
└── src/
    ├── main.rs
    ├── config.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── category.rs
    │   └── product.rs
    ├── repositories/
    │   ├── mod.rs
    │   ├── category_repo.rs
    │   └── product_repo.rs
    ├── services/
    │   ├── mod.rs
    │   ├── category_service.rs
    │   └── product_service.rs
    └── handlers/
        ├── mod.rs
        ├── categories.rs
        └── products.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "rest_api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono", "migrate"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
validator = { version = "0.18", features = ["derive"] }
env_logger = "0.11"
log = "0.4"
dotenv = "0.15"
```

---

## 3. Config

```rust
// src/config.rs
use std::env;

#[derive(Clone, Debug)]
pub struct Config {
    pub database_url: String,
    pub server_host: String,
    pub server_port: u16,
    pub jwt_secret: String,
}

impl Config {
    pub fn from_env() -> Self {
        dotenv::dotenv().ok();
        Self {
            database_url: env::var("DATABASE_URL")
                .expect("DATABASE_URL must be set"),
            server_host: env::var("SERVER_HOST")
                .unwrap_or_else(|_| "127.0.0.1".to_string()),
            server_port: env::var("SERVER_PORT")
                .unwrap_or_else(|_| "8080".to_string())
                .parse()
                .expect("SERVER_PORT must be a number"),
            jwt_secret: env::var("JWT_SECRET")
                .unwrap_or_else(|_| "secret".to_string()),
        }
    }
}
```

---

## 4. Errors

```rust
// src/errors.rs
use actix_web::{HttpResponse, ResponseError};
use serde::Serialize;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),

    #[error("Bad request: {0}")]
    BadRequest(String),

    #[error("Conflict: {0}")]
    Conflict(String),

    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("Validation error: {0}")]
    Validation(String),

    #[error("Internal server error")]
    Internal,
}

#[derive(Serialize)]
struct ErrorBody {
    success: bool,
    error: String,
    code: u16,
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        let (status, message) = match self {
            AppError::NotFound(msg) => (404, msg.clone()),
            AppError::BadRequest(msg) => (400, msg.clone()),
            AppError::Conflict(msg) => (409, msg.clone()),
            AppError::Validation(msg) => (422, msg.clone()),
            AppError::Database(e) => {
                log::error!("Database error: {}", e);
                (500, "Database error".to_string())
            }
            AppError::Internal => (500, "Internal server error".to_string()),
        };

        HttpResponse::build(
            actix_web::http::StatusCode::from_u16(status).unwrap()
        ).json(ErrorBody {
            success: false,
            error: message,
            code: status,
        })
    }
}

pub type AppResult<T> = Result<T, AppError>;
```

---

## 5. Models

```rust
// src/models/category.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use validator::Validate;

#[derive(Debug, Clone, Serialize, FromRow)]
pub struct Category {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateCategoryRequest {
    #[validate(length(min = 2, max = 100))]
    pub name: String,
    #[validate(length(max = 500))]
    pub description: Option<String>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateCategoryRequest {
    #[validate(length(min = 2, max = 100))]
    pub name: Option<String>,
    #[validate(length(max = 500))]
    pub description: Option<String>,
}
```

```rust
// src/models/product.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use validator::Validate;

#[derive(Debug, Clone, Serialize, FromRow)]
pub struct Product {
    pub id: Uuid,
    pub category_id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub price: f64,
    pub stock: i32,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct ProductWithCategory {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub price: f64,
    pub stock: i32,
    pub is_active: bool,
    pub category_name: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateProductRequest {
    pub category_id: Uuid,
    #[validate(length(min = 2, max = 200))]
    pub name: String,
    #[validate(length(max = 2000))]
    pub description: Option<String>,
    #[validate(range(min = 0.0))]
    pub price: f64,
    #[validate(range(min = 0))]
    pub stock: i32,
}

#[derive(Debug, Deserialize, Validate)]
pub struct UpdateProductRequest {
    pub category_id: Option<Uuid>,
    #[validate(length(min = 2, max = 200))]
    pub name: Option<String>,
    pub description: Option<String>,
    #[validate(range(min = 0.0))]
    pub price: Option<f64>,
    pub stock: Option<i32>,
    pub is_active: Option<bool>,
}

#[derive(Debug, Deserialize)]
pub struct ProductQuery {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub category_id: Option<Uuid>,
    pub search: Option<String>,
    pub min_price: Option<f64>,
    pub max_price: Option<f64>,
    pub in_stock: Option<bool>,
}

#[derive(Debug, Serialize)]
pub struct PaginatedProducts {
    pub data: Vec<ProductWithCategory>,
    pub total: i64,
    pub page: u32,
    pub per_page: u32,
    pub total_pages: u32,
}
```

---

## 6. Repositories

```rust
// src/repositories/category_repo.rs
use sqlx::PgPool;
use uuid::Uuid;
use crate::models::category::{Category, CreateCategoryRequest, UpdateCategoryRequest};
use crate::errors::AppResult;

pub struct CategoryRepository {
    pool: PgPool,
}

impl CategoryRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn find_all(&self) -> AppResult<Vec<Category>> {
        let categories = sqlx::query_as!(
            Category,
            "SELECT * FROM categories ORDER BY name"
        )
        .fetch_all(&self.pool)
        .await?;
        Ok(categories)
    }

    pub async fn find_by_id(&self, id: Uuid) -> AppResult<Option<Category>> {
        let category = sqlx::query_as!(
            Category,
            "SELECT * FROM categories WHERE id = $1",
            id
        )
        .fetch_optional(&self.pool)
        .await?;
        Ok(category)
    }

    pub async fn find_by_slug(&self, slug: &str) -> AppResult<Option<Category>> {
        let category = sqlx::query_as!(
            Category,
            "SELECT * FROM categories WHERE slug = $1",
            slug
        )
        .fetch_optional(&self.pool)
        .await?;
        Ok(category)
    }

    pub async fn create(&self, req: &CreateCategoryRequest) -> AppResult<Category> {
        let slug = slugify(&req.name);
        let category = sqlx::query_as!(
            Category,
            r#"INSERT INTO categories (name, slug, description)
               VALUES ($1, $2, $3)
               RETURNING *"#,
            req.name,
            slug,
            req.description
        )
        .fetch_one(&self.pool)
        .await?;
        Ok(category)
    }

    pub async fn update(&self, id: Uuid, req: &UpdateCategoryRequest) -> AppResult<Option<Category>> {
        let category = sqlx::query_as!(
            Category,
            r#"UPDATE categories
               SET name = COALESCE($2, name),
                   slug = COALESCE($3, slug),
                   description = COALESCE($4, description),
                   updated_at = NOW()
               WHERE id = $1
               RETURNING *"#,
            id,
            req.name,
            req.name.as_ref().map(|n| slugify(n)),
            req.description
        )
        .fetch_optional(&self.pool)
        .await?;
        Ok(category)
    }

    pub async fn delete(&self, id: Uuid) -> AppResult<bool> {
        let result = sqlx::query!(
            "DELETE FROM categories WHERE id = $1",
            id
        )
        .execute(&self.pool)
        .await?;
        Ok(result.rows_affected() > 0)
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

```rust
// src/repositories/product_repo.rs
use sqlx::PgPool;
use uuid::Uuid;
use crate::models::product::{
    Product, ProductWithCategory, CreateProductRequest,
    UpdateProductRequest, ProductQuery, PaginatedProducts
};
use crate::errors::AppResult;

pub struct ProductRepository {
    pool: PgPool,
}

impl ProductRepository {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }

    pub async fn find_all(&self, query: &ProductQuery) -> AppResult<PaginatedProducts> {
        let page = query.page.unwrap_or(1).max(1);
        let per_page = query.per_page.unwrap_or(20).min(100).max(1);
        let offset = ((page - 1) * per_page) as i64;

        // Count query
        let total: i64 = sqlx::query_scalar!(
            r#"SELECT COUNT(*) FROM products p
               WHERE ($1::uuid IS NULL OR p.category_id = $1)
               AND ($2::text IS NULL OR p.name ILIKE '%' || $2 || '%')
               AND ($3::float8 IS NULL OR p.price >= $3)
               AND ($4::float8 IS NULL OR p.price <= $4)
               AND ($5::bool IS NULL OR ($5 = true AND p.stock > 0) OR ($5 = false))"#,
            query.category_id,
            query.search,
            query.min_price,
            query.max_price,
            query.in_stock
        )
        .fetch_one(&self.pool)
        .await?
        .unwrap_or(0);

        // Data query
        let products = sqlx::query_as!(
            ProductWithCategory,
            r#"SELECT p.id, p.name, p.slug, p.description, p.price,
                      p.stock, p.is_active, c.name as category_name, p.created_at
               FROM products p
               JOIN categories c ON p.category_id = c.id
               WHERE ($1::uuid IS NULL OR p.category_id = $1)
               AND ($2::text IS NULL OR p.name ILIKE '%' || $2 || '%')
               AND ($3::float8 IS NULL OR p.price >= $3)
               AND ($4::float8 IS NULL OR p.price <= $4)
               AND ($5::bool IS NULL OR ($5 = true AND p.stock > 0) OR ($5 = false))
               ORDER BY p.created_at DESC
               LIMIT $6 OFFSET $7"#,
            query.category_id,
            query.search,
            query.min_price,
            query.max_price,
            query.in_stock,
            per_page as i64,
            offset
        )
        .fetch_all(&self.pool)
        .await?;

        let total_pages = ((total as u32 + per_page - 1) / per_page).max(1);

        Ok(PaginatedProducts {
            data: products,
            total,
            page,
            per_page,
            total_pages,
        })
    }

    pub async fn find_by_id(&self, id: Uuid) -> AppResult<Option<ProductWithCategory>> {
        let product = sqlx::query_as!(
            ProductWithCategory,
            r#"SELECT p.id, p.name, p.slug, p.description, p.price,
                      p.stock, p.is_active, c.name as category_name, p.created_at
               FROM products p
               JOIN categories c ON p.category_id = c.id
               WHERE p.id = $1"#,
            id
        )
        .fetch_optional(&self.pool)
        .await?;
        Ok(product)
    }

    pub async fn create(&self, req: &CreateProductRequest) -> AppResult<Product> {
        let slug = slugify(&req.name);
        let product = sqlx::query_as!(
            Product,
            r#"INSERT INTO products (category_id, name, slug, description, price, stock)
               VALUES ($1, $2, $3, $4, $5, $6)
               RETURNING *"#,
            req.category_id,
            req.name,
            slug,
            req.description,
            req.price,
            req.stock
        )
        .fetch_one(&self.pool)
        .await?;
        Ok(product)
    }

    pub async fn update(&self, id: Uuid, req: &UpdateProductRequest) -> AppResult<Option<Product>> {
        let product = sqlx::query_as!(
            Product,
            r#"UPDATE products
               SET category_id = COALESCE($2, category_id),
                   name = COALESCE($3, name),
                   slug = COALESCE($4, slug),
                   description = COALESCE($5, description),
                   price = COALESCE($6, price),
                   stock = COALESCE($7, stock),
                   is_active = COALESCE($8, is_active),
                   updated_at = NOW()
               WHERE id = $1
               RETURNING *"#,
            id,
            req.category_id,
            req.name,
            req.name.as_ref().map(|n| slugify(n)),
            req.description,
            req.price,
            req.stock,
            req.is_active
        )
        .fetch_optional(&self.pool)
        .await?;
        Ok(product)
    }

    pub async fn delete(&self, id: Uuid) -> AppResult<bool> {
        let result = sqlx::query!("DELETE FROM products WHERE id = $1", id)
            .execute(&self.pool)
            .await?;
        Ok(result.rows_affected() > 0)
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

## 7. Handlers

```rust
// src/handlers/categories.rs
use actix_web::{delete, get, post, put, web, HttpResponse};
use uuid::Uuid;
use validator::Validate;
use crate::errors::{AppError, AppResult};
use crate::models::category::{CreateCategoryRequest, UpdateCategoryRequest};
use crate::repositories::category_repo::CategoryRepository;

#[get("/categories")]
async fn list_categories(
    repo: web::Data<CategoryRepository>,
) -> AppResult<HttpResponse> {
    let categories = repo.find_all().await?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": categories
    })))
}

#[get("/categories/{id}")]
async fn get_category(
    path: web::Path<Uuid>,
    repo: web::Data<CategoryRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    let category = repo.find_by_id(id).await?
        .ok_or_else(|| AppError::NotFound(format!("Category {} not found", id)))?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": category
    })))
}

#[post("/categories")]
async fn create_category(
    body: web::Json<CreateCategoryRequest>,
    repo: web::Data<CategoryRepository>,
) -> AppResult<HttpResponse> {
    body.validate().map_err(|e| AppError::Validation(e.to_string()))?;

    // Check duplicate slug
    let slug = body.name.to_lowercase().replace(' ', "-");
    if repo.find_by_slug(&slug).await?.is_some() {
        return Err(AppError::Conflict(format!("Category '{}' already exists", body.name)));
    }

    let category = repo.create(&body).await?;
    Ok(HttpResponse::Created().json(serde_json::json!({
        "success": true,
        "data": category
    })))
}

#[put("/categories/{id}")]
async fn update_category(
    path: web::Path<Uuid>,
    body: web::Json<UpdateCategoryRequest>,
    repo: web::Data<CategoryRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    body.validate().map_err(|e| AppError::Validation(e.to_string()))?;

    let category = repo.update(id, &body).await?
        .ok_or_else(|| AppError::NotFound(format!("Category {} not found", id)))?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": category
    })))
}

#[delete("/categories/{id}")]
async fn delete_category(
    path: web::Path<Uuid>,
    repo: web::Data<CategoryRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    let deleted = repo.delete(id).await?;
    if !deleted {
        return Err(AppError::NotFound(format!("Category {} not found", id)));
    }
    Ok(HttpResponse::NoContent().finish())
}

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(list_categories)
       .service(get_category)
       .service(create_category)
       .service(update_category)
       .service(delete_category);
}
```

```rust
// src/handlers/products.rs
use actix_web::{delete, get, post, put, web, HttpResponse};
use uuid::Uuid;
use validator::Validate;
use crate::errors::{AppError, AppResult};
use crate::models::product::{CreateProductRequest, UpdateProductRequest, ProductQuery};
use crate::repositories::product_repo::ProductRepository;

#[get("/products")]
async fn list_products(
    query: web::Query<ProductQuery>,
    repo: web::Data<ProductRepository>,
) -> AppResult<HttpResponse> {
    let result = repo.find_all(&query).await?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": result.data,
        "meta": {
            "total": result.total,
            "page": result.page,
            "per_page": result.per_page,
            "total_pages": result.total_pages
        }
    })))
}

#[get("/products/{id}")]
async fn get_product(
    path: web::Path<Uuid>,
    repo: web::Data<ProductRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    let product = repo.find_by_id(id).await?
        .ok_or_else(|| AppError::NotFound(format!("Product {} not found", id)))?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": product
    })))
}

#[post("/products")]
async fn create_product(
    body: web::Json<CreateProductRequest>,
    repo: web::Data<ProductRepository>,
) -> AppResult<HttpResponse> {
    body.validate().map_err(|e| AppError::Validation(e.to_string()))?;
    let product = repo.create(&body).await?;
    Ok(HttpResponse::Created().json(serde_json::json!({
        "success": true,
        "data": product
    })))
}

#[put("/products/{id}")]
async fn update_product(
    path: web::Path<Uuid>,
    body: web::Json<UpdateProductRequest>,
    repo: web::Data<ProductRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    body.validate().map_err(|e| AppError::Validation(e.to_string()))?;
    let product = repo.update(id, &body).await?
        .ok_or_else(|| AppError::NotFound(format!("Product {} not found", id)))?;
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "data": product
    })))
}

#[delete("/products/{id}")]
async fn delete_product(
    path: web::Path<Uuid>,
    repo: web::Data<ProductRepository>,
) -> AppResult<HttpResponse> {
    let id = path.into_inner();
    let deleted = repo.delete(id).await?;
    if !deleted {
        return Err(AppError::NotFound(format!("Product {} not found", id)));
    }
    Ok(HttpResponse::NoContent().finish())
}

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(list_products)
       .service(get_product)
       .service(create_product)
       .service(update_product)
       .service(delete_product);
}
```

---

## 8. Main.rs

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware::Logger};
use sqlx::postgres::PgPoolOptions;

mod config;
mod errors;
mod models;
mod repositories;
mod handlers;

use config::Config;
use repositories::{
    category_repo::CategoryRepository,
    product_repo::ProductRepository,
};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    let config = Config::from_env();

    // Database pool
    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect(&config.database_url)
        .await
        .expect("Failed to connect to database");

    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");

    log::info!("Starting server at {}:{}", config.server_host, config.server_port);

    let category_repo = web::Data::new(CategoryRepository::new(pool.clone()));
    let product_repo = web::Data::new(ProductRepository::new(pool.clone()));

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .app_data(category_repo.clone())
            .app_data(product_repo.clone())
            .service(
                web::scope("/api/v1")
                    .configure(handlers::categories::configure)
                    .configure(handlers::products::configure)
            )
    })
    .bind(format!("{}:{}", config.server_host, config.server_port))?
    .run()
    .await
}
```

---

## 9. Migrations

```sql
-- migrations/20240101_001_create_categories.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE categories (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name        VARCHAR(100) NOT NULL UNIQUE,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_categories_slug ON categories(slug);
```

```sql
-- migrations/20240101_002_create_products.sql
CREATE TABLE products (
    id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
    name        VARCHAR(200) NOT NULL,
    slug        VARCHAR(200) NOT NULL UNIQUE,
    description TEXT,
    price       DOUBLE PRECISION NOT NULL DEFAULT 0,
    stock       INTEGER NOT NULL DEFAULT 0,
    is_active   BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_slug ON products(slug);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_name_search ON products USING gin(to_tsvector('english', name));
```

---

## 10. ทดสอบ API

```bash
# Health check
curl http://localhost:8080/api/v1/categories

# Create category
curl -X POST http://localhost:8080/api/v1/categories \
  -H "Content-Type: application/json" \
  -d '{"name": "Electronics", "description": "Electronic products"}'

# Create product
curl -X POST http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{
    "category_id": "UUID_HERE",
    "name": "iPhone 15",
    "price": 35000.00,
    "stock": 100
  }'

# List products with filters
curl "http://localhost:8080/api/v1/products?page=1&per_page=10&min_price=1000&in_stock=true"

# Search products
curl "http://localhost:8080/api/v1/products?search=iPhone"
```

---

## 11. สรุปสิ่งที่เรียนรู้

✅ Clean architecture (Handler → Repository)  
✅ CRUD ครบถ้วน (GET/POST/PUT/DELETE)  
✅ Pagination + filtering + search  
✅ Input validation ด้วย validator crate  
✅ Error handling ด้วย thiserror + ResponseError  
✅ Database migrations  
✅ Dynamic SQL queries  
✅ Proper HTTP status codes  

---

*[← Part 039: Query Optimization](../part_039/README.md) | [Part 041: JWT Authentication →](../part_041/README.md)*
