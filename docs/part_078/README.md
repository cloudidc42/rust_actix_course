# Part 078: API Documentation with OpenAPI

## บทนำ

OpenAPI (เดิมชื่อ Swagger) เป็น specification มาตรฐานสำหรับ documenting REST APIs บทนี้จะครอบคลุมการใช้ `utoipa` crate เพื่อสร้าง OpenAPI documentation อัตโนมัติจาก Rust code

## 1. utoipa Crate สำหรับ OpenAPI 3

### 1.1 Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
utoipa = { version = "4", features = ["actix_extras"] }
utoipa-swagger-ui = { version = "6", features = ["actix-web"] }
utoipa-redoc = { version = "3", features = ["actix-web"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-rustls", "macros"] }
```

### 1.2 Basic OpenAPI Setup

```rust
// src/main.rs
use actix_web::{web, App, HttpServer};
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;

// กำหนด OpenAPI specification
#[derive(OpenApi)]
#[openapi(
    info(
        title = "My Rust API",
        description = "API documentation สำหรับ My Rust Application",
        version = "1.0.0",
        contact(
            name = "Development Team",
            email = "dev@example.com",
            url = "https://example.com"
        ),
        license(
            name = "MIT",
            url = "https://opensource.org/licenses/MIT"
        )
    ),
    servers(
        (url = "https://api.example.com/v1", description = "Production"),
        (url = "https://staging.example.com/v1", description = "Staging"),
        (url = "http://localhost:8080/api/v1", description = "Development")
    ),
    paths(
        handlers::products::list_products,
        handlers::products::get_product,
        handlers::products::create_product,
        handlers::products::update_product,
        handlers::products::delete_product,
        handlers::users::register,
        handlers::users::login,
        handlers::users::get_profile,
    ),
    components(
        schemas(
            models::Product,
            models::CreateProduct,
            models::UpdateProduct,
            models::User,
            models::LoginRequest,
            models::LoginResponse,
            models::ErrorResponse,
            models::PaginationMeta,
            models::ProductListResponse,
        ),
        security_schemes(
            ("bearer_auth" = (
                type = Http,
                scheme = "bearer",
                bearer_format = "JWT",
                description = "JWT Token สำหรับ Authentication"
            ))
        )
    ),
    tags(
        (name = "products", description = "Product management endpoints"),
        (name = "users", description = "User management endpoints"),
        (name = "auth", description = "Authentication endpoints"),
    ),
    external_docs(
        url = "https://docs.example.com",
        description = "เอกสารเพิ่มเติม"
    )
)]
pub struct ApiDoc;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let openapi = ApiDoc::openapi();
    
    HttpServer::new(move || {
        App::new()
            // Swagger UI
            .service(
                SwaggerUi::new("/docs/{_:.*}")
                    .url("/api-docs/openapi.json", openapi.clone())
            )
            // API routes
            .service(
                web::scope("/api/v1")
                    .configure(handlers::products::configure)
                    .configure(handlers::users::configure)
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

## 2. #[utoipa::path] Annotations

### 2.1 GET Endpoint

```rust
// src/handlers/products.rs
use actix_web::{web, HttpResponse};
use utoipa;
use crate::models::*;

/// รายการสินค้าทั้งหมด
///
/// ดึงรายการสินค้าแบบ pagination พร้อม filter ตาม category
#[utoipa::path(
    get,
    path = "/api/v1/products",
    tag = "products",
    params(
        ("category" = Option<String>, Query, description = "กรองตาม category"),
        ("page" = Option<i32>, Query, description = "หมายเลขหน้า (เริ่มจาก 1)", example = 1),
        ("per_page" = Option<i32>, Query, description = "จำนวนรายการต่อหน้า (สูงสุด 100)", example = 20),
        ("sort" = Option<String>, Query, description = "ฟิลด์ที่ใช้เรียง (name, price, created_at)"),
        ("order" = Option<String>, Query, description = "ทิศทางการเรียง (asc, desc)"),
    ),
    responses(
        (status = 200, description = "รายการสินค้าที่ดึงมาสำเร็จ", body = ProductListResponse),
        (status = 400, description = "ค่า parameter ไม่ถูกต้อง", body = ErrorResponse),
        (status = 500, description = "เกิดข้อผิดพลาดภายใน server", body = ErrorResponse),
    ),
    security(
        // endpoint นี้ไม่ต้อง auth
    )
)]
pub async fn list_products(
    query: web::Query<ProductQuery>,
    db: web::Data<sqlx::PgPool>,
) -> HttpResponse {
    // implementation...
    HttpResponse::Ok().json(serde_json::json!({ "data": [] }))
}

/// ดูข้อมูลสินค้าตาม ID
#[utoipa::path(
    get,
    path = "/api/v1/products/{id}",
    tag = "products",
    params(
        ("id" = i32, Path, description = "ID ของสินค้า", example = 1),
    ),
    responses(
        (status = 200, description = "ข้อมูลสินค้า", body = Product),
        (status = 404, description = "ไม่พบสินค้า", body = ErrorResponse,
            example = json!({"code": "NOT_FOUND", "message": "Product not found"})),
        (status = 500, description = "เกิดข้อผิดพลาดภายใน server", body = ErrorResponse),
    )
)]
pub async fn get_product(
    path: web::Path<i32>,
    db: web::Data<sqlx::PgPool>,
) -> HttpResponse {
    // implementation...
    HttpResponse::Ok().json(serde_json::json!({}))
}
```

### 2.2 POST Endpoint

```rust
/// สร้างสินค้าใหม่
#[utoipa::path(
    post,
    path = "/api/v1/products",
    tag = "products",
    request_body(
        content = CreateProduct,
        description = "ข้อมูลสินค้าใหม่",
        content_type = "application/json",
        example = json!({
            "name": "iPhone 15 Pro",
            "price": 42900.00,
            "stock": 100,
            "category": "electronics",
            "description": "สมาร์ทโฟน Apple รุ่นล่าสุด",
            "tags": ["smartphone", "apple", "ios"]
        })
    ),
    responses(
        (status = 201, description = "สร้างสินค้าสำเร็จ", body = Product),
        (status = 400, description = "ข้อมูลไม่ถูกต้อง", body = ErrorResponse),
        (status = 401, description = "ต้อง authenticate ก่อน", body = ErrorResponse),
        (status = 403, description = "ไม่มีสิทธิ์", body = ErrorResponse),
        (status = 409, description = "สินค้าชื่อนี้มีอยู่แล้ว", body = ErrorResponse),
        (status = 422, description = "ข้อมูลไม่ผ่าน validation", body = ValidationErrorResponse),
        (status = 500, description = "เกิดข้อผิดพลาดภายใน server", body = ErrorResponse),
    ),
    security(
        ("bearer_auth" = [])
    )
)]
pub async fn create_product(
    product: web::Json<CreateProduct>,
    db: web::Data<sqlx::PgPool>,
) -> HttpResponse {
    // implementation...
    HttpResponse::Created().json(serde_json::json!({ "id": 1 }))
}

/// อัพเดทข้อมูลสินค้า
#[utoipa::path(
    put,
    path = "/api/v1/products/{id}",
    tag = "products",
    params(
        ("id" = i32, Path, description = "ID ของสินค้าที่ต้องการอัพเดท"),
    ),
    request_body = UpdateProduct,
    responses(
        (status = 200, description = "อัพเดทสำเร็จ", body = Product),
        (status = 400, description = "ข้อมูลไม่ถูกต้อง", body = ErrorResponse),
        (status = 401, description = "ต้อง authenticate ก่อน", body = ErrorResponse),
        (status = 404, description = "ไม่พบสินค้า", body = ErrorResponse),
    ),
    security(
        ("bearer_auth" = [])
    )
)]
pub async fn update_product(
    path: web::Path<i32>,
    product: web::Json<UpdateProduct>,
    db: web::Data<sqlx::PgPool>,
) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({}))
}

/// ลบสินค้า
#[utoipa::path(
    delete,
    path = "/api/v1/products/{id}",
    tag = "products",
    params(
        ("id" = i32, Path, description = "ID ของสินค้าที่ต้องการลบ"),
    ),
    responses(
        (status = 204, description = "ลบสำเร็จ"),
        (status = 401, description = "ต้อง authenticate ก่อน", body = ErrorResponse),
        (status = 403, description = "ไม่มีสิทธิ์ลบสินค้านี้", body = ErrorResponse),
        (status = 404, description = "ไม่พบสินค้า", body = ErrorResponse),
    ),
    security(
        ("bearer_auth" = [])
    )
)]
pub async fn delete_product(
    path: web::Path<i32>,
    db: web::Data<sqlx::PgPool>,
) -> HttpResponse {
    HttpResponse::NoContent().finish()
}
```

## 3. Schema Generation

### 3.1 Basic Schema

```rust
// src/models.rs
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

/// ข้อมูลสินค้า
#[derive(Debug, Serialize, Deserialize, ToSchema, Clone)]
pub struct Product {
    /// ID ของสินค้า
    #[schema(example = 1)]
    pub id: i32,
    
    /// ชื่อสินค้า
    #[schema(example = "iPhone 15 Pro")]
    pub name: String,
    
    /// ราคาสินค้า (บาท)
    #[schema(example = 42900.0)]
    pub price: f64,
    
    /// จำนวนสินค้าในคลัง
    #[schema(example = 100)]
    pub stock: i32,
    
    /// หมวดหมู่สินค้า
    #[schema(example = "electronics")]
    pub category: String,
    
    /// รายละเอียดสินค้า
    #[schema(example = "สมาร์ทโฟน Apple รุ่นล่าสุด")]
    pub description: Option<String>,
    
    /// แท็กสินค้า
    #[schema(example = json!(["smartphone", "apple"]))]
    pub tags: Vec<String>,
    
    /// วันที่สร้าง
    #[schema(value_type = String, format = DateTime, example = "2024-01-01T00:00:00Z")]
    pub created_at: chrono::DateTime<chrono::Utc>,
    
    /// วันที่อัพเดทล่าสุด
    #[schema(value_type = String, format = DateTime, example = "2024-01-01T00:00:00Z")]
    pub updated_at: chrono::DateTime<chrono::Utc>,
}

/// ข้อมูลสำหรับสร้างสินค้าใหม่
#[derive(Debug, Deserialize, ToSchema)]
pub struct CreateProduct {
    /// ชื่อสินค้า (ต้อง unique)
    #[schema(example = "iPhone 15 Pro", min_length = 1, max_length = 255)]
    pub name: String,
    
    /// ราคาสินค้า (ต้องมากกว่า 0)
    #[schema(example = 42900.0, minimum = 0.0)]
    pub price: f64,
    
    /// จำนวนสินค้าในคลัง
    #[schema(example = 100, minimum = 0)]
    pub stock: i32,
    
    /// หมวดหมู่สินค้า
    #[schema(example = "electronics")]
    pub category: String,
    
    /// รายละเอียดสินค้า (optional)
    #[schema(example = "สมาร์ทโฟน Apple รุ่นล่าสุด")]
    pub description: Option<String>,
    
    /// แท็กสินค้า
    #[schema(example = json!(["smartphone", "apple"]))]
    #[serde(default)]
    pub tags: Vec<String>,
}

/// ข้อมูลสำหรับอัพเดทสินค้า (ทุก field เป็น optional)
#[derive(Debug, Deserialize, ToSchema)]
pub struct UpdateProduct {
    #[schema(example = "iPhone 15 Pro Max")]
    pub name: Option<String>,
    
    #[schema(example = 49900.0)]
    pub price: Option<f64>,
    
    #[schema(example = 50)]
    pub stock: Option<i32>,
    
    pub description: Option<String>,
    
    #[schema(example = json!(["smartphone", "apple", "pro"]))]
    pub tags: Option<Vec<String>>,
}
```

### 3.2 Advanced Schema Types

```rust
// src/models/responses.rs
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

/// Response สำหรับรายการสินค้า
#[derive(Debug, Serialize, ToSchema)]
pub struct ProductListResponse {
    /// รายการสินค้า
    pub data: Vec<Product>,
    
    /// ข้อมูล pagination
    pub meta: PaginationMeta,
}

/// ข้อมูล Pagination
#[derive(Debug, Serialize, ToSchema)]
pub struct PaginationMeta {
    /// หน้าปัจจุบัน
    #[schema(example = 1)]
    pub current_page: i32,
    
    /// จำนวนรายการต่อหน้า
    #[schema(example = 20)]
    pub per_page: i32,
    
    /// จำนวนรายการทั้งหมด
    #[schema(example = 150)]
    pub total_items: i64,
    
    /// จำนวนหน้าทั้งหมด
    #[schema(example = 8)]
    pub total_pages: i32,
    
    /// มีหน้าถัดไปหรือไม่
    #[schema(example = true)]
    pub has_next: bool,
    
    /// มีหน้าก่อนหน้าหรือไม่
    #[schema(example = false)]
    pub has_prev: bool,
}

/// Error Response
#[derive(Debug, Serialize, ToSchema)]
pub struct ErrorResponse {
    /// Error code
    #[schema(example = "VALIDATION_ERROR")]
    pub code: String,
    
    /// ข้อความ error
    #[schema(example = "ข้อมูลที่ส่งมาไม่ถูกต้อง")]
    pub message: String,
    
    /// รายละเอียดเพิ่มเติม
    pub details: Option<serde_json::Value>,
}

/// Validation Error Response
#[derive(Debug, Serialize, ToSchema)]
pub struct ValidationErrorResponse {
    pub code: String,
    pub message: String,
    /// รายการ field ที่ validation ไม่ผ่าน
    pub fields: Vec<FieldError>,
}

/// ข้อมูล validation error ของ field
#[derive(Debug, Serialize, ToSchema)]
pub struct FieldError {
    /// ชื่อ field
    #[schema(example = "price")]
    pub field: String,
    
    /// ข้อความ error
    #[schema(example = "ราคาต้องมากกว่า 0")]
    pub message: String,
}

/// Enum สำหรับ product status
#[derive(Debug, Serialize, Deserialize, ToSchema)]
#[serde(rename_all = "lowercase")]
pub enum ProductStatus {
    /// สินค้าพร้อมขาย
    Active,
    /// สินค้าหมดสต็อก
    OutOfStock,
    /// สินค้าถูกระงับ
    Suspended,
}
```

## 4. Swagger UI Integration

### 4.1 Setup Swagger UI

```rust
// src/main.rs
use actix_web::{web, App, HttpServer};
use utoipa::OpenApi;
use utoipa_swagger_ui::SwaggerUi;
use utoipa_redoc::{Redoc, Servable};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let openapi = ApiDoc::openapi();
    let openapi_clone = openapi.clone();
    
    HttpServer::new(move || {
        App::new()
            // Swagger UI - interactive documentation
            .service(
                SwaggerUi::new("/docs/swagger/{_:.*}")
                    .url("/api-docs/openapi.json", openapi.clone())
                    .config(
                        utoipa_swagger_ui::Config::default()
                            .display_request_duration(true)
                            .filter(true)
                            .syntax_highlight(true)
                            .try_it_out_enabled(true)
                            .supported_submit_methods(vec!["get", "post", "put", "delete"])
                    )
            )
            // Redoc - cleaner documentation
            .service(
                Redoc::with_url("/docs/redoc", openapi_clone.clone())
            )
            // Raw OpenAPI JSON
            .route(
                "/api-docs/openapi.json",
                web::get().to(move || {
                    let spec = openapi_clone.clone();
                    async move {
                        actix_web::HttpResponse::Ok()
                            .content_type("application/json")
                            .json(spec)
                    }
                })
            )
            // API routes
            .service(
                web::scope("/api/v1")
                    .configure(handlers::products::configure)
                    .configure(handlers::users::configure)
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

### 4.2 Custom Swagger UI Config

```rust
// Custom Swagger UI configuration
let swagger_config = utoipa_swagger_ui::Config::default()
    // UI customization
    .display_request_duration(true)
    .filter(true)
    .deep_linking(true)
    .default_models_expand_depth(2)
    .default_model_expand_depth(3)
    .doc_expansion("none")  // หรือ "list", "full"
    .operations_sorter("alpha")
    .show_extensions(true)
    .show_common_extensions(true)
    .try_it_out_enabled(true)
    // Syntax highlighting
    .syntax_highlight(true)
    // Custom CSS
    .with_credentials(false);
```

## 5. Authentication ใน Docs

### 5.1 JWT Bearer Authentication

```rust
// กำหนด security scheme ใน ApiDoc
#[derive(OpenApi)]
#[openapi(
    components(
        security_schemes(
            ("bearer_auth" = (
                type = Http,
                scheme = "bearer",
                bearer_format = "JWT",
                description = "
                    JWT Bearer Token authentication
                    
                    วิธีใช้:
                    1. เรียก POST /api/v1/auth/login เพื่อรับ token
                    2. ใส่ token ในรูปแบบ: `Bearer <your-token>`
                    
                    Example: `Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...`
                "
            )),
            ("api_key" = (
                type = ApiKey,
                name = "X-API-Key",
                in = "header",
                description = "API Key authentication สำหรับ server-to-server"
            ))
        )
    )
)]
pub struct ApiDoc;

// ใน handler annotations
#[utoipa::path(
    post,
    path = "/api/v1/products",
    tag = "products",
    security(
        ("bearer_auth" = []),      // ต้องใช้ JWT
    )
)]
pub async fn create_product() -> HttpResponse {
    todo!()
}

// Endpoint ที่รองรับหลาย auth methods
#[utoipa::path(
    get,
    path = "/api/v1/admin/stats",
    tag = "admin",
    security(
        ("bearer_auth" = ["admin"]),   // ต้องมี admin scope
        ("api_key" = []),              // หรือ API key ก็ได้
    )
)]
pub async fn get_admin_stats() -> HttpResponse {
    todo!()
}
```

### 5.2 Login Endpoint

```rust
/// เข้าสู่ระบบ
#[utoipa::path(
    post,
    path = "/api/v1/auth/login",
    tag = "auth",
    request_body(
        content = LoginRequest,
        description = "ข้อมูล login",
        examples(
            ("user" = (
                summary = "Login เป็น user ปกติ",
                value = json!({
                    "email": "user@example.com",
                    "password": "password123"
                })
            )),
            ("admin" = (
                summary = "Login เป็น admin",
                value = json!({
                    "email": "admin@example.com",
                    "password": "admin123"
                })
            ))
        )
    ),
    responses(
        (status = 200, description = "Login สำเร็จ", body = LoginResponse,
            example = json!({
                "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
                "expires_at": "2024-12-31T23:59:59Z",
                "user": {
                    "id": 1,
                    "email": "user@example.com",
                    "name": "Test User"
                }
            })
        ),
        (status = 401, description = "Email หรือ password ไม่ถูกต้อง", body = ErrorResponse),
        (status = 429, description = "Login บ่อยเกินไป", body = ErrorResponse,
            headers(
                ("Retry-After" = String, description = "วินาทีที่ต้องรอก่อน retry")
            )
        ),
    )
)]
pub async fn login(body: web::Json<LoginRequest>) -> HttpResponse {
    // implementation
    HttpResponse::Ok().json(serde_json::json!({}))
}
```

## 6. Request/Response Examples

### 6.1 Multiple Examples

```rust
// src/models/users.rs
use serde::{Deserialize, Serialize};
use utoipa::ToSchema;

#[derive(Debug, Deserialize, ToSchema)]
pub struct LoginRequest {
    #[schema(example = "user@example.com", format = Email)]
    pub email: String,
    
    #[schema(example = "password123", min_length = 8, format = Password)]
    pub password: String,
    
    /// จดจำการ login (30 วัน)
    #[schema(default = false)]
    #[serde(default)]
    pub remember_me: bool,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct LoginResponse {
    /// JWT Access Token
    #[schema(example = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...")]
    pub access_token: String,
    
    /// JWT Refresh Token (ถ้า remember_me = true)
    pub refresh_token: Option<String>,
    
    /// ประเภทของ token
    #[schema(example = "Bearer")]
    pub token_type: String,
    
    /// วันหมดอายุ (Unix timestamp)
    #[schema(example = 1735689600)]
    pub expires_in: i64,
    
    /// ข้อมูล user
    pub user: UserProfile,
}

#[derive(Debug, Serialize, ToSchema)]
pub struct UserProfile {
    #[schema(example = 1)]
    pub id: i32,
    
    #[schema(example = "user@example.com")]
    pub email: String,
    
    #[schema(example = "John Doe")]
    pub name: String,
    
    #[schema(example = json!(["user", "moderator"]))]
    pub roles: Vec<String>,
    
    #[schema(value_type = String, format = DateTime)]
    pub created_at: chrono::DateTime<chrono::Utc>,
}
```

### 6.2 Inline Examples ใน Paths

```rust
#[utoipa::path(
    post,
    path = "/api/v1/orders",
    tag = "orders",
    request_body(
        content = CreateOrderRequest,
        examples(
            ("simple" = (
                summary = "Order ง่ายๆ",
                description = "Order สินค้า 1 ชิ้น",
                value = json!({
                    "items": [
                        {"product_id": 1, "quantity": 2}
                    ],
                    "shipping_address": "123 ถนนสุขุมวิท กรุงเทพ",
                    "payment_method": "credit_card"
                })
            )),
            ("multiple" = (
                summary = "Order หลายรายการ",
                description = "Order สินค้าหลายชิ้นพร้อมกัน",
                value = json!({
                    "items": [
                        {"product_id": 1, "quantity": 2},
                        {"product_id": 5, "quantity": 1},
                        {"product_id": 10, "quantity": 3}
                    ],
                    "shipping_address": "456 ถนนพระราม 4 กรุงเทพ",
                    "payment_method": "bank_transfer",
                    "discount_code": "SAVE10"
                })
            ))
        )
    ),
    responses(
        (status = 201, description = "สร้าง order สำเร็จ", body = OrderResponse),
        (status = 400, description = "ข้อมูลไม่ถูกต้อง", body = ErrorResponse),
        (status = 422, description = "Validation error", body = ValidationErrorResponse),
    ),
    security(("bearer_auth" = []))
)]
pub async fn create_order() -> HttpResponse {
    HttpResponse::Created().finish()
}
```

## 7. API Changelog

### 7.1 Version Documentation

```rust
// src/api_versions.rs - Document API versions

/// API Version 2.0 - Breaking Changes
/// 
/// ## Breaking Changes in v2
/// 
/// 1. Product responses now include `tags` array (was `tag` string)
/// 2. Pagination uses `page`/`per_page` instead of `offset`/`limit`  
/// 3. All dates now in ISO 8601 format
/// 
/// ## New Features in v2
/// 
/// - Product variants support
/// - Bulk operations endpoints
/// - WebSocket notifications
/// - GraphQL endpoint
/// 
/// ## Deprecated (will be removed in v3)
/// 
/// - `GET /api/v1/products` - use `GET /api/v2/products`
/// - `POST /api/v1/auth/token` - use `POST /api/v2/auth/login`
#[derive(OpenApi)]
#[openapi(
    info(
        title = "My API v2",
        version = "2.0.0",
        description = "
            # My API
            
            ## Changelog
            
            ### v2.0.0 (2024-01-01)
            - Breaking: Product tags now array instead of string
            - Breaking: Pagination parameters changed
            - New: Product variants
            - New: Bulk endpoints
            
            ### v1.5.0 (2023-10-01)
            - New: Order tracking
            - Fixed: Rate limiting headers
            
            ### v1.0.0 (2023-01-01)
            - Initial release
        "
    )
)]
pub struct ApiDocV2;
```

## 8. Practical: Fully Documented API

### 8.1 Complete Handler File

```rust
// src/handlers/products.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use utoipa;
use sqlx::PgPool;

use crate::models::{
    Product, CreateProduct, UpdateProduct,
    ProductListResponse, PaginationMeta,
    ErrorResponse,
};

#[derive(Debug, Deserialize, utoipa::IntoParams)]
pub struct ProductQuery {
    /// กรองตาม category
    pub category: Option<String>,
    
    /// หน้าที่ต้องการ (เริ่มจาก 1)
    #[param(minimum = 1, example = 1)]
    pub page: Option<i64>,
    
    /// จำนวนรายการต่อหน้า (สูงสุด 100)
    #[param(minimum = 1, maximum = 100, example = 20)]
    pub per_page: Option<i64>,
    
    /// ราคาต่ำสุด
    #[param(minimum = 0)]
    pub min_price: Option<f64>,
    
    /// ราคาสูงสุด
    pub max_price: Option<f64>,
    
    /// ค้นหาตามชื่อสินค้า
    pub search: Option<String>,
}

/// รายการสินค้า
#[utoipa::path(
    get,
    path = "/api/v1/products",
    tag = "products",
    params(ProductQuery),
    responses(
        (
            status = 200,
            description = "ดึงรายการสินค้าสำเร็จ",
            body = ProductListResponse,
            headers(
                ("X-Total-Count" = i64, description = "จำนวนสินค้าทั้งหมด"),
                ("X-Page-Count" = i32, description = "จำนวนหน้าทั้งหมด"),
            )
        ),
        (status = 400, description = "Parameter ไม่ถูกต้อง", body = ErrorResponse),
        (status = 500, description = "Server error", body = ErrorResponse),
    )
)]
pub async fn list_products(
    query: web::Query<ProductQuery>,
    db: web::Data<PgPool>,
) -> HttpResponse {
    let page = query.page.unwrap_or(1).max(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let offset = (page - 1) * per_page;
    
    let total: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM products WHERE ($1::text IS NULL OR category = $1)",
        query.category.as_deref()
    )
    .fetch_one(db.get_ref())
    .await
    .unwrap_or(Some(0))
    .unwrap_or(0);
    
    let products = sqlx::query_as!(
        Product,
        r#"
        SELECT id, name, price, stock, category, description, 
               '[]'::json as "tags!: Vec<String>",
               created_at, updated_at
        FROM products
        WHERE ($1::text IS NULL OR category = $1)
          AND ($2::float8 IS NULL OR price >= $2)
          AND ($3::float8 IS NULL OR price <= $3)
          AND ($4::text IS NULL OR name ILIKE '%' || $4 || '%')
        ORDER BY id
        LIMIT $5 OFFSET $6
        "#,
        query.category.as_deref(),
        query.min_price,
        query.max_price,
        query.search.as_deref(),
        per_page,
        offset
    )
    .fetch_all(db.get_ref())
    .await
    .unwrap_or_default();
    
    let total_pages = ((total as f64) / (per_page as f64)).ceil() as i32;
    
    HttpResponse::Ok()
        .insert_header(("X-Total-Count", total.to_string()))
        .insert_header(("X-Page-Count", total_pages.to_string()))
        .json(ProductListResponse {
            data: products,
            meta: PaginationMeta {
                current_page: page as i32,
                per_page: per_page as i32,
                total_items: total,
                total_pages,
                has_next: page < total_pages as i64,
                has_prev: page > 1,
            },
        })
}

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/products")
            .route("", web::get().to(list_products))
            .route("", web::post().to(create_product))
            .route("/{id}", web::get().to(get_product))
            .route("/{id}", web::put().to(update_product))
            .route("/{id}", web::delete().to(delete_product))
    );
}
```

### 8.2 Generate Static Documentation

```bash
#!/bin/bash
# scripts/generate-docs.sh

# Generate OpenAPI spec
cargo run --bin export-openapi > openapi.json

# Validate spec
npx @redocly/cli lint openapi.json

# Generate static HTML docs
npx @redocly/cli build-docs openapi.json -o docs/index.html

# Generate Markdown docs
npx widdershins openapi.json -o docs/api.md

# Bundle เพื่อ deploy
echo "API documentation generated successfully"
```

```rust
// src/bin/export-openapi.rs
fn main() {
    let doc = ApiDoc::openapi();
    println!("{}", doc.to_pretty_json().unwrap());
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **utoipa** - สร้าง OpenAPI spec จาก Rust code อัตโนมัติ
2. **#[utoipa::path]** - annotate endpoints
3. **ToSchema** - สร้าง schema จาก structs
4. **Swagger UI** - interactive documentation
5. **Authentication** - JWT และ API Key ใน docs
6. **Examples** - request/response examples
7. **Versioning** - document API changes
8. **Complete API** - fully documented handlers

---

[⬅️ Part 077: Database Backup and Recovery](../part_077/README.md) | [➡️ Part 079: Testing Strategies](../part_079/README.md)
