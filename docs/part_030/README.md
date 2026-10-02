# Part 030: Query Builder and Pagination ใน Actix-web

## สารบัญ
- [แนะนำ Query Builder](#แนะนำ-query-builder)
- [Pagination Query Parameters](#pagination-query-parameters)
- [Cursor-Based Pagination](#cursor-based-pagination)
- [Offset-Based Pagination](#offset-based-pagination)
- [Sorting Parameters](#sorting-parameters)
- [Filtering](#filtering)
- [Search](#search)
- [Building Dynamic SQL with sqlx](#building-dynamic-sql-with-sqlx)
- [Response Metadata](#response-metadata)

---

## แนะนำ Query Builder

Query Builder คือ pattern สำหรับสร้าง SQL queries อย่าง dynamic ตาม parameters ที่ได้รับ ช่วยให้เราสามารถ filter, sort, paginate ข้อมูลได้อย่างยืดหยุ่น

### Dependencies

```toml
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
base64 = "0.21"
```

---

## Pagination Query Parameters

การออกแบบ query parameters สำหรับ pagination

```rust
use serde::{Deserialize, Serialize};
use actix_web::{web, HttpResponse, Result};

// Basic pagination parameters
#[derive(Debug, Deserialize)]
struct PaginationParams {
    // Offset pagination
    page: Option<u64>,
    per_page: Option<u64>,
    
    // Cursor pagination
    cursor: Option<String>,
    limit: Option<u64>,
    
    // Sorting
    sort_by: Option<String>,
    sort_order: Option<SortOrder>,
    
    // Filtering
    search: Option<String>,
    status: Option<String>,
    category_id: Option<u64>,
    
    // Date range
    created_after: Option<String>,
    created_before: Option<String>,
}

#[derive(Debug, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "lowercase")]
enum SortOrder {
    Asc,
    Desc,
}

impl Default for SortOrder {
    fn default() -> Self {
        SortOrder::Desc
    }
}

impl SortOrder {
    fn as_sql(&self) -> &'static str {
        match self {
            SortOrder::Asc => "ASC",
            SortOrder::Desc => "DESC",
        }
    }
}

// Normalized pagination params
#[derive(Debug, Clone)]
struct NormalizedPagination {
    page: u64,
    per_page: u64,
    offset: u64,
}

impl NormalizedPagination {
    fn from_params(page: Option<u64>, per_page: Option<u64>) -> Self {
        let page = page.unwrap_or(1).max(1);
        let per_page = per_page.unwrap_or(20).min(100).max(1);
        let offset = (page - 1) * per_page;
        
        NormalizedPagination { page, per_page, offset }
    }
}

// Comprehensive query parameters
#[derive(Debug, Deserialize)]
struct ProductQueryParams {
    // Pagination
    #[serde(default = "default_page")]
    page: u64,
    
    #[serde(default = "default_per_page")]
    per_page: u64,
    
    // Sorting
    #[serde(default = "default_sort_by")]
    sort_by: String,
    
    #[serde(default)]
    sort_order: SortOrder,
    
    // Filtering
    search: Option<String>,
    category: Option<String>,
    min_price: Option<f64>,
    max_price: Option<f64>,
    in_stock: Option<bool>,
    featured: Option<bool>,
    
    // Fields selection
    fields: Option<String>,  // comma-separated: "id,name,price"
}

fn default_page() -> u64 { 1 }
fn default_per_page() -> u64 { 20 }
fn default_sort_by() -> String { "created_at".to_string() }

impl ProductQueryParams {
    fn normalized_page(&self) -> u64 { self.page.max(1) }
    fn normalized_per_page(&self) -> u64 { self.per_page.min(100).max(1) }
    fn offset(&self) -> u64 { (self.normalized_page() - 1) * self.normalized_per_page() }
    
    fn validate_sort_by(&self) -> Result<String, String> {
        let allowed = ["id", "name", "price", "created_at", "updated_at", "views"];
        if allowed.contains(&self.sort_by.as_str()) {
            Ok(self.sort_by.clone())
        } else {
            Err(format!("Invalid sort_by '{}'. Allowed: {:?}", self.sort_by, allowed))
        }
    }
}

// Response structures
#[derive(Serialize)]
struct PaginationMeta {
    total: u64,
    page: u64,
    per_page: u64,
    total_pages: u64,
    has_next: bool,
    has_prev: bool,
    next_page: Option<u64>,
    prev_page: Option<u64>,
    // Links
    links: PaginationLinks,
}

#[derive(Serialize)]
struct PaginationLinks {
    first: String,
    last: String,
    prev: Option<String>,
    next: Option<String>,
    self_: String,
}

impl PaginationMeta {
    fn new(total: u64, page: u64, per_page: u64, base_url: &str) -> Self {
        let total_pages = if per_page > 0 { (total + per_page - 1) / per_page } else { 1 };
        let has_next = page < total_pages;
        let has_prev = page > 1;
        
        PaginationMeta {
            total,
            page,
            per_page,
            total_pages,
            has_next,
            has_prev,
            next_page: if has_next { Some(page + 1) } else { None },
            prev_page: if has_prev { Some(page - 1) } else { None },
            links: PaginationLinks {
                first: format!("{}?page=1&per_page={}", base_url, per_page),
                last: format!("{}?page={}&per_page={}", base_url, total_pages, per_page),
                prev: if has_prev { Some(format!("{}?page={}&per_page={}", base_url, page - 1, per_page)) } else { None },
                next: if has_next { Some(format!("{}?page={}&per_page={}", base_url, page + 1, per_page)) } else { None },
                self_: format!("{}?page={}&per_page={}", base_url, page, per_page),
            },
        }
    }
}

#[derive(Serialize)]
struct PaginatedResponse<T: Serialize> {
    data: Vec<T>,
    meta: PaginationMeta,
    filters: serde_json::Value,
}
```

---

## Cursor-Based Pagination

Cursor pagination เหมาะสำหรับ real-time data ที่เพิ่มขึ้นเรื่อยๆ

```rust
use serde::{Deserialize, Serialize};
use base64::{Engine as _, engine::general_purpose};
use chrono::{DateTime, Utc};
use uuid::Uuid;

// Cursor encoding/decoding
#[derive(Debug, Serialize, Deserialize)]
struct Cursor {
    id: Option<Uuid>,
    created_at: Option<DateTime<Utc>>,
    value: Option<serde_json::Value>,  // สำหรับ custom cursor
}

impl Cursor {
    fn encode(&self) -> String {
        let json = serde_json::to_string(self).unwrap_or_default();
        general_purpose::URL_SAFE_NO_PAD.encode(json.as_bytes())
    }
    
    fn decode(encoded: &str) -> Option<Self> {
        let bytes = general_purpose::URL_SAFE_NO_PAD.decode(encoded).ok()?;
        let json = String::from_utf8(bytes).ok()?;
        serde_json::from_str(&json).ok()
    }
    
    fn from_uuid(id: Uuid) -> Self {
        Cursor { id: Some(id), created_at: None, value: None }
    }
    
    fn from_timestamp(created_at: DateTime<Utc>, id: Uuid) -> Self {
        Cursor { id: Some(id), created_at: Some(created_at), value: None }
    }
}

// Cursor pagination params
#[derive(Debug, Deserialize)]
struct CursorParams {
    after: Option<String>,     // cursor สำหรับ next page
    before: Option<String>,    // cursor สำหรับ prev page  
    limit: Option<u64>,
    sort_by: Option<String>,
}

// Cursor pagination response
#[derive(Serialize)]
struct CursorPaginatedResponse<T: Serialize> {
    data: Vec<T>,
    page_info: PageInfo,
    total_count: Option<u64>,  // optional เพราะ count query อาจช้า
}

#[derive(Serialize)]
struct PageInfo {
    start_cursor: Option<String>,
    end_cursor: Option<String>,
    has_next_page: bool,
    has_previous_page: bool,
    count: usize,
}

// Product model
#[derive(Debug, Serialize, sqlx::FromRow)]
struct Product {
    id: Uuid,
    name: String,
    price: f64,
    category: String,
    is_active: bool,
    created_at: DateTime<Utc>,
}

// Handler ที่ใช้ cursor pagination
use actix_web::{web, HttpResponse, Result};
use sqlx::PgPool;

async fn list_products_cursor(
    pool: web::Data<PgPool>,
    query: web::Query<CursorParams>,
) -> Result<HttpResponse> {
    let limit = query.limit.unwrap_or(20).min(100) as i64;
    let fetch_limit = limit + 1;  // fetch one extra to check if has_next
    
    let products = if let Some(after) = &query.after {
        // Next page
        if let Some(cursor) = Cursor::decode(after) {
            if let (Some(created_at), Some(id)) = (cursor.created_at, cursor.id) {
                sqlx::query_as!(
                    Product,
                    r#"
                    SELECT id, name, price, category, is_active, created_at
                    FROM products
                    WHERE is_active = true
                    AND (created_at, id) < ($1, $2)
                    ORDER BY created_at DESC, id DESC
                    LIMIT $3
                    "#,
                    created_at,
                    id,
                    fetch_limit
                )
                .fetch_all(pool.get_ref())
                .await
                .unwrap_or_default()
            } else {
                vec![]
            }
        } else {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Invalid cursor"
            })));
        }
    } else if let Some(before) = &query.before {
        // Previous page
        if let Some(cursor) = Cursor::decode(before) {
            if let (Some(created_at), Some(id)) = (cursor.created_at, cursor.id) {
                let mut products = sqlx::query_as!(
                    Product,
                    r#"
                    SELECT id, name, price, category, is_active, created_at
                    FROM products
                    WHERE is_active = true
                    AND (created_at, id) > ($1, $2)
                    ORDER BY created_at ASC, id ASC
                    LIMIT $3
                    "#,
                    created_at,
                    id,
                    fetch_limit
                )
                .fetch_all(pool.get_ref())
                .await
                .unwrap_or_default();
                
                products.reverse();  // reverse เพื่อให้ order ถูกต้อง
                products
            } else {
                vec![]
            }
        } else {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Invalid cursor"
            })));
        }
    } else {
        // First page
        sqlx::query_as!(
            Product,
            r#"
            SELECT id, name, price, category, is_active, created_at
            FROM products
            WHERE is_active = true
            ORDER BY created_at DESC, id DESC
            LIMIT $1
            "#,
            fetch_limit
        )
        .fetch_all(pool.get_ref())
        .await
        .unwrap_or_default()
    };
    
    let has_next = products.len() > limit as usize;
    let mut data = products;
    if has_next {
        data.pop();
    }
    
    let has_prev = query.after.is_some() || (query.before.is_some() && !data.is_empty());
    
    let start_cursor = data.first().map(|p| {
        Cursor::from_timestamp(p.created_at, p.id).encode()
    });
    
    let end_cursor = data.last().map(|p| {
        Cursor::from_timestamp(p.created_at, p.id).encode()
    });
    
    let count = data.len();
    
    Ok(HttpResponse::Ok().json(CursorPaginatedResponse {
        data,
        page_info: PageInfo {
            start_cursor,
            end_cursor,
            has_next_page: has_next,
            has_previous_page: has_prev,
            count,
        },
        total_count: None,  // ไม่ count เพื่อประสิทธิภาพ
    }))
}
```

---

## Offset-Based Pagination

Offset pagination เป็นรูปแบบที่นิยมและเข้าใจง่าย

```rust
use sqlx::PgPool;
use actix_web::{web, HttpResponse, Result};
use serde::{Deserialize, Serialize};

// User model
#[derive(Debug, Serialize, sqlx::FromRow)]
struct User {
    id: i64,
    username: String,
    email: String,
    role: String,
    is_active: bool,
    created_at: chrono::DateTime<chrono::Utc>,
}

// User query parameters
#[derive(Debug, Deserialize)]
struct UserListParams {
    page: Option<i64>,
    per_page: Option<i64>,
    sort_by: Option<String>,
    sort_order: Option<String>,
    search: Option<String>,
    role: Option<String>,
    is_active: Option<bool>,
    created_after: Option<String>,
    created_before: Option<String>,
}

impl UserListParams {
    fn page(&self) -> i64 { self.page.unwrap_or(1).max(1) }
    fn per_page(&self) -> i64 { self.per_page.unwrap_or(20).min(100).max(1) }
    fn offset(&self) -> i64 { (self.page() - 1) * self.per_page() }
    
    fn sort_by(&self) -> &str {
        match self.sort_by.as_deref() {
            Some("username") => "username",
            Some("email") => "email",
            Some("role") => "role",
            Some("created_at") => "created_at",
            _ => "created_at",
        }
    }
    
    fn sort_order(&self) -> &str {
        match self.sort_order.as_deref() {
            Some("asc") => "ASC",
            _ => "DESC",
        }
    }
}

// Handler ที่ใช้ offset pagination
async fn list_users(
    pool: web::Data<PgPool>,
    query: web::Query<UserListParams>,
) -> Result<HttpResponse> {
    let params = query.into_inner();
    
    // Validate sort_by
    let sort_by = params.sort_by();
    let sort_order = params.sort_order();
    let limit = params.per_page();
    let offset = params.offset();
    
    // สร้าง dynamic WHERE clause
    let mut conditions: Vec<String> = vec![];
    let mut bind_index = 1i32;
    
    if params.search.is_some() {
        conditions.push(format!(
            "(username ILIKE ${} OR email ILIKE ${})",
            bind_index, bind_index + 1
        ));
        bind_index += 2;
    }
    
    if params.role.is_some() {
        conditions.push(format!("role = ${}", bind_index));
        bind_index += 1;
    }
    
    if params.is_active.is_some() {
        conditions.push(format!("is_active = ${}", bind_index));
        bind_index += 1;
    }
    
    let where_clause = if conditions.is_empty() {
        String::new()
    } else {
        format!("WHERE {}", conditions.join(" AND "))
    };
    
    // เนื่องจาก sqlx ไม่รองรับ dynamic queries ตรงๆ เราต้องใช้ query_builder
    // หรือใช้ sqlx::query!() กับ conditions ที่ compile-time known
    
    // ตัวอย่างใช้ query_builder
    let users_result = fetch_users_dynamic(
        &pool,
        &params,
        sort_by,
        sort_order,
        limit,
        offset,
    ).await;
    
    let total_result = count_users_dynamic(&pool, &params).await;
    
    match (users_result, total_result) {
        (Ok(users), Ok(total)) => {
            let page = params.page();
            let per_page = params.per_page();
            let total_pages = (total + per_page - 1) / per_page;
            
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "data": users,
                "meta": {
                    "total": total,
                    "page": page,
                    "per_page": per_page,
                    "total_pages": total_pages,
                    "has_next": page < total_pages,
                    "has_prev": page > 1,
                    "next_page": if page < total_pages { Some(page + 1) } else { None },
                    "prev_page": if page > 1 { Some(page - 1) } else { None }
                },
                "filters": {
                    "search": params.search,
                    "role": params.role,
                    "is_active": params.is_active,
                    "sort_by": sort_by,
                    "sort_order": sort_order
                }
            })))
        },
        _ => Ok(HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Failed to fetch users"
        })))
    }
}

async fn fetch_users_dynamic(
    pool: &PgPool,
    params: &UserListParams,
    sort_by: &str,
    sort_order: &str,
    limit: i64,
    offset: i64,
) -> Result<Vec<User>, sqlx::Error> {
    // ใช้ sqlx query_builder
    let mut builder = sqlx::QueryBuilder::<sqlx::Postgres>::new(
        "SELECT id, username, email, role, is_active, created_at FROM users WHERE 1=1"
    );
    
    if let Some(ref search) = params.search {
        let search_pattern = format!("%{}%", search);
        builder.push(" AND (username ILIKE ");
        builder.push_bind(search_pattern.clone());
        builder.push(" OR email ILIKE ");
        builder.push_bind(search_pattern);
        builder.push(")");
    }
    
    if let Some(ref role) = params.role {
        builder.push(" AND role = ");
        builder.push_bind(role);
    }
    
    if let Some(active) = params.is_active {
        builder.push(" AND is_active = ");
        builder.push_bind(active);
    }
    
    // Sort (ต้องใช้ push เพราะ sort column ไม่ใช่ bind parameter)
    // ต้อง validate sort_by ก่อนเพื่อป้องกัน SQL injection!
    builder.push(format!(" ORDER BY {} {}", sort_by, sort_order));
    
    builder.push(" LIMIT ");
    builder.push_bind(limit);
    builder.push(" OFFSET ");
    builder.push_bind(offset);
    
    builder.build_query_as::<User>()
        .fetch_all(pool)
        .await
}

async fn count_users_dynamic(
    pool: &PgPool,
    params: &UserListParams,
) -> Result<i64, sqlx::Error> {
    let mut builder = sqlx::QueryBuilder::<sqlx::Postgres>::new(
        "SELECT COUNT(*) FROM users WHERE 1=1"
    );
    
    if let Some(ref search) = params.search {
        let search_pattern = format!("%{}%", search);
        builder.push(" AND (username ILIKE ");
        builder.push_bind(search_pattern.clone());
        builder.push(" OR email ILIKE ");
        builder.push_bind(search_pattern);
        builder.push(")");
    }
    
    if let Some(ref role) = params.role {
        builder.push(" AND role = ");
        builder.push_bind(role);
    }
    
    if let Some(active) = params.is_active {
        builder.push(" AND is_active = ");
        builder.push_bind(active);
    }
    
    let row: (i64,) = builder.build_query_as()
        .fetch_one(pool)
        .await?;
    
    Ok(row.0)
}
```

---

## Sorting Parameters

การจัดการ sorting อย่างปลอดภัย

```rust
use std::collections::HashMap;

// Sortable fields whitelist
struct SortableFields {
    fields: HashMap<&'static str, &'static str>,  // alias -> column
}

impl SortableFields {
    fn for_products() -> Self {
        let mut fields = HashMap::new();
        fields.insert("id", "products.id");
        fields.insert("name", "products.name");
        fields.insert("price", "products.price");
        fields.insert("created_at", "products.created_at");
        fields.insert("updated_at", "products.updated_at");
        fields.insert("views", "products.views");
        fields.insert("category", "categories.name");
        
        SortableFields { fields }
    }
    
    fn for_users() -> Self {
        let mut fields = HashMap::new();
        fields.insert("id", "users.id");
        fields.insert("username", "users.username");
        fields.insert("email", "users.email");
        fields.insert("created_at", "users.created_at");
        fields.insert("last_login", "users.last_login_at");
        
        SortableFields { fields }
    }
    
    fn validate(&self, field: &str) -> Option<&str> {
        self.fields.get(field).copied()
    }
    
    fn validate_with_default(&self, field: Option<&str>, default: &'static str) -> &str {
        field
            .and_then(|f| self.validate(f))
            .unwrap_or_else(|| self.fields.get(default).copied().unwrap_or("id"))
    }
}

// Multi-column sort
#[derive(Debug, Deserialize)]
struct MultiSortParams {
    sort: Option<String>,  // "name:asc,price:desc"
}

#[derive(Debug)]
struct SortSpec {
    column: String,
    order: SortOrder,
}

impl MultiSortParams {
    fn parse_sort(&self, sortable: &SortableFields) -> Vec<SortSpec> {
        let sort_str = match &self.sort {
            Some(s) => s,
            None => return vec![SortSpec {
                column: "created_at".to_string(),
                order: SortOrder::Desc,
            }],
        };
        
        sort_str.split(',')
            .filter_map(|part| {
                let parts: Vec<&str> = part.splitn(2, ':').collect();
                if parts.is_empty() {
                    return None;
                }
                
                let field = parts[0].trim();
                let order = if parts.len() > 1 {
                    match parts[1].trim().to_lowercase().as_str() {
                        "asc" => SortOrder::Asc,
                        _ => SortOrder::Desc,
                    }
                } else {
                    SortOrder::Desc
                };
                
                // Validate field name
                sortable.validate(field).map(|col| SortSpec {
                    column: col.to_string(),
                    order,
                })
            })
            .take(3)  // max 3 sort columns
            .collect()
    }
}

// สร้าง ORDER BY clause
fn build_order_by(sorts: &[SortSpec]) -> String {
    if sorts.is_empty() {
        return "ORDER BY created_at DESC".to_string();
    }
    
    let parts: Vec<String> = sorts.iter()
        .map(|s| format!("{} {}", s.column, s.order.as_sql()))
        .collect();
    
    format!("ORDER BY {}", parts.join(", "))
}
```

---

## Filtering

การสร้าง filter system ที่ยืดหยุ่น

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

// Filter operator
#[derive(Debug, Deserialize, Clone)]
#[serde(rename_all = "lowercase")]
enum FilterOp {
    Eq,
    Ne,
    Gt,
    Gte,
    Lt,
    Lte,
    Like,
    ILike,  // case-insensitive LIKE
    In,
    NotIn,
    IsNull,
    IsNotNull,
    Between,
}

// Filter value
#[derive(Debug, Deserialize, Clone)]
#[serde(untagged)]
enum FilterValue {
    String(String),
    Number(f64),
    Bool(bool),
    Array(Vec<serde_json::Value>),
    Null,
}

// Filter definition
#[derive(Debug, Deserialize)]
struct FilterParam {
    field: String,
    op: FilterOp,
    value: Option<FilterValue>,
}

// Product filters
#[derive(Debug, Deserialize)]
struct ProductFilters {
    // Price range
    min_price: Option<f64>,
    max_price: Option<f64>,
    
    // Category
    category: Option<String>,
    category_ids: Option<Vec<i64>>,
    
    // Status
    is_active: Option<bool>,
    is_featured: Option<bool>,
    in_stock: Option<bool>,
    
    // Text search
    name: Option<String>,
    sku: Option<String>,
    
    // Date range
    created_after: Option<chrono::DateTime<chrono::Utc>>,
    created_before: Option<chrono::DateTime<chrono::Utc>>,
    
    // Custom attributes (JSON filter)
    attributes: Option<HashMap<String, serde_json::Value>>,
}

impl ProductFilters {
    fn is_empty(&self) -> bool {
        self.min_price.is_none()
            && self.max_price.is_none()
            && self.category.is_none()
            && self.category_ids.is_none()
            && self.is_active.is_none()
            && self.is_featured.is_none()
            && self.in_stock.is_none()
            && self.name.is_none()
            && self.sku.is_none()
            && self.created_after.is_none()
            && self.created_before.is_none()
    }
    
    fn to_active_filters(&self) -> serde_json::Value {
        let mut active = serde_json::Map::new();
        
        if let Some(v) = self.min_price { active.insert("min_price".to_string(), v.into()); }
        if let Some(v) = self.max_price { active.insert("max_price".to_string(), v.into()); }
        if let Some(ref v) = self.category { active.insert("category".to_string(), v.clone().into()); }
        if let Some(v) = self.is_active { active.insert("is_active".to_string(), v.into()); }
        if let Some(v) = self.is_featured { active.insert("is_featured".to_string(), v.into()); }
        if let Some(ref v) = self.name { active.insert("name".to_string(), v.clone().into()); }
        
        serde_json::Value::Object(active)
    }
}

// Apply filters ด้วย QueryBuilder
async fn fetch_products_with_filters(
    pool: &PgPool,
    filters: &ProductFilters,
    pagination: &NormalizedPagination,
    sort_by: &str,
    sort_order: &str,
) -> Result<(Vec<serde_json::Value>, i64), sqlx::Error> {
    let mut query_builder = sqlx::QueryBuilder::<sqlx::Postgres>::new(
        "SELECT p.id, p.name, p.price, p.sku, p.is_active, p.created_at, c.name as category_name
         FROM products p
         LEFT JOIN categories c ON p.category_id = c.id
         WHERE 1=1"
    );
    
    let mut count_builder = sqlx::QueryBuilder::<sqlx::Postgres>::new(
        "SELECT COUNT(*) FROM products p LEFT JOIN categories c ON p.category_id = c.id WHERE 1=1"
    );
    
    // Apply filters to both queries
    let apply_filters = |builder: &mut sqlx::QueryBuilder<sqlx::Postgres>| {
        if let Some(min_price) = filters.min_price {
            builder.push(" AND p.price >= ").push_bind(min_price);
        }
        
        if let Some(max_price) = filters.max_price {
            builder.push(" AND p.price <= ").push_bind(max_price);
        }
        
        if let Some(ref category) = filters.category {
            builder.push(" AND c.name ILIKE ").push_bind(format!("%{}%", category));
        }
        
        if let Some(ref ids) = filters.category_ids {
            if !ids.is_empty() {
                builder.push(" AND p.category_id = ANY(");
                builder.push_bind(ids.as_slice());
                builder.push(")");
            }
        }
        
        if let Some(active) = filters.is_active {
            builder.push(" AND p.is_active = ").push_bind(active);
        }
        
        if let Some(featured) = filters.is_featured {
            builder.push(" AND p.is_featured = ").push_bind(featured);
        }
        
        if let Some(in_stock) = filters.in_stock {
            if in_stock {
                builder.push(" AND p.stock_quantity > 0");
            } else {
                builder.push(" AND p.stock_quantity = 0");
            }
        }
        
        if let Some(ref name) = filters.name {
            builder.push(" AND p.name ILIKE ").push_bind(format!("%{}%", name));
        }
        
        if let Some(ref sku) = filters.sku {
            builder.push(" AND p.sku = ").push_bind(sku);
        }
        
        if let Some(created_after) = filters.created_after {
            builder.push(" AND p.created_at >= ").push_bind(created_after);
        }
        
        if let Some(created_before) = filters.created_before {
            builder.push(" AND p.created_at <= ").push_bind(created_before);
        }
    };
    
    apply_filters(&mut query_builder);
    apply_filters(&mut count_builder);
    
    // Count
    let count: (i64,) = count_builder.build_query_as()
        .fetch_one(pool)
        .await?;
    
    // Paginated query
    query_builder.push(format!(" ORDER BY p.{} {}", sort_by, sort_order));
    query_builder.push(" LIMIT ").push_bind(pagination.per_page as i64);
    query_builder.push(" OFFSET ").push_bind(pagination.offset as i64);
    
    let rows = query_builder.build()
        .fetch_all(pool)
        .await?;
    
    // Map rows to JSON (simplified)
    let products: Vec<serde_json::Value> = rows.iter()
        .map(|row| {
            use sqlx::Row;
            serde_json::json!({
                "id": row.get::<i64, _>("id"),
                "name": row.get::<String, _>("name"),
                "price": row.get::<f64, _>("price"),
            })
        })
        .collect();
    
    Ok((products, count.0))
}
```

---

## Search

Full-text search และ search patterns

```rust
use sqlx::PgPool;
use actix_web::{web, HttpResponse, Result};
use serde::{Deserialize, Serialize};

// Search parameters
#[derive(Debug, Deserialize)]
struct SearchParams {
    q: String,              // search query
    fields: Option<String>, // comma-separated fields to search
    page: Option<i64>,
    per_page: Option<i64>,
    highlight: Option<bool>,  // ส่ง highlight ของผลลัพธ์กลับมา
}

// Search result with highlight
#[derive(Debug, Serialize)]
struct SearchResult {
    id: i64,
    title: String,
    content: String,
    relevance_score: f64,
    highlights: Option<Vec<Highlight>>,
}

#[derive(Debug, Serialize)]
struct Highlight {
    field: String,
    fragments: Vec<String>,
}

// Full-text search handler
async fn full_text_search(
    pool: web::Data<PgPool>,
    query: web::Query<SearchParams>,
) -> Result<HttpResponse> {
    let params = query.into_inner();
    
    if params.q.trim().is_empty() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Search query cannot be empty"
        })));
    }
    
    // Sanitize search query
    let search_query = sanitize_search_query(&params.q);
    
    let page = params.page.unwrap_or(1).max(1);
    let per_page = params.per_page.unwrap_or(20).min(50);
    let offset = (page - 1) * per_page;
    
    // PostgreSQL full-text search
    // to_tsvector: แปลง text เป็น tsvector
    // to_tsquery: แปลง search term เป็น tsquery
    // ts_rank: คำนวณ relevance score
    // ts_headline: สร้าง highlight
    
    let results = sqlx::query!(
        r#"
        SELECT
            id,
            title,
            content,
            ts_rank(
                to_tsvector('english', title || ' ' || content),
                to_tsquery('english', $1)
            ) as relevance_score,
            ts_headline(
                'english',
                title || ' ' || content,
                to_tsquery('english', $1),
                'MaxWords=35, MinWords=15, ShortWord=3, HighlightAll=FALSE, MaxFragments=3'
            ) as headline
        FROM articles
        WHERE
            to_tsvector('english', title || ' ' || content) @@ to_tsquery('english', $1)
            AND is_published = true
        ORDER BY relevance_score DESC
        LIMIT $2 OFFSET $3
        "#,
        search_query,
        per_page,
        offset
    )
    .fetch_all(pool.get_ref())
    .await;
    
    let total_count = sqlx::query_scalar!(
        r#"
        SELECT COUNT(*)
        FROM articles
        WHERE
            to_tsvector('english', title || ' ' || content) @@ to_tsquery('english', $1)
            AND is_published = true
        "#,
        search_query
    )
    .fetch_one(pool.get_ref())
    .await;
    
    match (results, total_count) {
        (Ok(rows), Ok(total)) => {
            let total = total.unwrap_or(0);
            let total_pages = (total + per_page - 1) / per_page;
            
            let items: Vec<serde_json::Value> = rows.iter().map(|row| {
                serde_json::json!({
                    "id": row.id,
                    "title": row.title,
                    "content": &row.content[..200.min(row.content.len())],
                    "relevance_score": row.relevance_score,
                    "headline": row.headline
                })
            }).collect();
            
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "data": items,
                "meta": {
                    "query": params.q,
                    "total": total,
                    "page": page,
                    "per_page": per_page,
                    "total_pages": total_pages
                }
            })))
        },
        _ => Ok(HttpResponse::InternalServerError().json(serde_json::json!({
            "error": "Search failed"
        })))
    }
}

// Simple ILIKE search (สำหรับ database ที่ไม่มี full-text search)
async fn simple_search(
    pool: web::Data<PgPool>,
    query: web::Query<SearchParams>,
) -> Result<HttpResponse> {
    let params = query.into_inner();
    let search_pattern = format!("%{}%", params.q.trim());
    let page = params.page.unwrap_or(1).max(1);
    let per_page = params.per_page.unwrap_or(20).min(50);
    let offset = (page - 1) * per_page;
    
    let users = sqlx::query!(
        r#"
        SELECT id, username, email, created_at
        FROM users
        WHERE
            username ILIKE $1
            OR email ILIKE $1
        ORDER BY
            CASE WHEN username ILIKE $1 THEN 0 ELSE 1 END,
            username
        LIMIT $2 OFFSET $3
        "#,
        search_pattern,
        per_page,
        offset
    )
    .fetch_all(pool.get_ref())
    .await;
    
    match users {
        Ok(rows) => {
            let items: Vec<serde_json::Value> = rows.iter().map(|row| {
                serde_json::json!({
                    "id": row.id,
                    "username": row.username,
                    "email": row.email,
                    "created_at": row.created_at
                })
            }).collect();
            
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "data": items,
                "query": params.q
            })))
        },
        Err(e) => {
            log::error!("Search error: {}", e);
            Ok(HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Search failed"
            })))
        }
    }
}

fn sanitize_search_query(query: &str) -> String {
    // แปลง search query สำหรับ PostgreSQL tsquery
    let words: Vec<&str> = query.split_whitespace()
        .filter(|w| w.len() >= 2)
        .take(10)
        .collect();
    
    words.join(" & ")
}
```

---

## Building Dynamic SQL with sqlx

การสร้าง SQL queries แบบ dynamic ด้วย sqlx QueryBuilder

```rust
use sqlx::{PgPool, QueryBuilder};
use serde::Deserialize;

// Complete example ของ QueryBuilder
#[derive(Debug, Deserialize)]
struct ArticleQuery {
    page: Option<i64>,
    per_page: Option<i64>,
    search: Option<String>,
    author_id: Option<i64>,
    tag: Option<String>,
    status: Option<String>,
    sort_by: Option<String>,
    sort_order: Option<String>,
    min_views: Option<i64>,
    from_date: Option<chrono::NaiveDate>,
    to_date: Option<chrono::NaiveDate>,
}

async fn dynamic_article_query(
    pool: &PgPool,
    params: &ArticleQuery,
) -> Result<(Vec<serde_json::Value>, i64), sqlx::Error> {
    let page = params.page.unwrap_or(1).max(1);
    let per_page = params.per_page.unwrap_or(20).min(100);
    let offset = (page - 1) * per_page;
    
    // Validate sort column
    let sort_col = match params.sort_by.as_deref() {
        Some("title") => "a.title",
        Some("views") => "a.views",
        Some("published_at") => "a.published_at",
        Some("created_at") | _ => "a.created_at",
    };
    
    let sort_dir = match params.sort_order.as_deref() {
        Some("asc") => "ASC",
        _ => "DESC",
    };
    
    // Base query
    let base_select = r#"
        SELECT
            a.id,
            a.title,
            a.slug,
            a.excerpt,
            a.status,
            a.views,
            a.published_at,
            a.created_at,
            u.username as author_username,
            COALESCE(array_agg(t.name ORDER BY t.name) FILTER (WHERE t.name IS NOT NULL), '{}') as tags
        FROM articles a
        JOIN users u ON a.author_id = u.id
        LEFT JOIN article_tags at ON a.id = at.article_id
        LEFT JOIN tags t ON at.tag_id = t.id
        WHERE 1=1
    "#;
    
    let mut query = QueryBuilder::<sqlx::Postgres>::new(base_select);
    let mut count_query = QueryBuilder::<sqlx::Postgres>::new(
        "SELECT COUNT(DISTINCT a.id) FROM articles a JOIN users u ON a.author_id = u.id LEFT JOIN article_tags at ON a.id = at.article_id LEFT JOIN tags t ON at.tag_id = t.id WHERE 1=1"
    );
    
    // Apply filters
    let mut apply_common_filters = |qb: &mut QueryBuilder<sqlx::Postgres>| {
        if let Some(ref search) = params.search {
            let pattern = format!("%{}%", search);
            qb.push(" AND (a.title ILIKE ")
              .push_bind(pattern.clone())
              .push(" OR a.content ILIKE ")
              .push_bind(pattern)
              .push(")");
        }
        
        if let Some(author_id) = params.author_id {
            qb.push(" AND a.author_id = ").push_bind(author_id);
        }
        
        if let Some(ref tag) = params.tag {
            qb.push(" AND EXISTS (SELECT 1 FROM article_tags at2 JOIN tags t2 ON at2.tag_id = t2.id WHERE at2.article_id = a.id AND t2.name = ")
              .push_bind(tag)
              .push(")");
        }
        
        if let Some(ref status) = params.status {
            qb.push(" AND a.status = ").push_bind(status);
        } else {
            qb.push(" AND a.status = 'published'");
        }
        
        if let Some(min_views) = params.min_views {
            qb.push(" AND a.views >= ").push_bind(min_views);
        }
        
        if let Some(from_date) = params.from_date {
            qb.push(" AND a.created_at::date >= ").push_bind(from_date);
        }
        
        if let Some(to_date) = params.to_date {
            qb.push(" AND a.created_at::date <= ").push_bind(to_date);
        }
    };
    
    apply_common_filters(&mut query);
    apply_common_filters(&mut count_query);
    
    // Count
    let count_row: (i64,) = count_query
        .build_query_as()
        .fetch_one(pool)
        .await?;
    let total = count_row.0;
    
    // Main query with GROUP BY, ORDER BY, LIMIT, OFFSET
    query.push(" GROUP BY a.id, u.username");
    query.push(format!(" ORDER BY {} {}", sort_col, sort_dir));
    query.push(" LIMIT ").push_bind(per_page);
    query.push(" OFFSET ").push_bind(offset);
    
    let rows = query.build().fetch_all(pool).await?;
    
    // Map to JSON
    let articles: Vec<serde_json::Value> = rows.iter().map(|row| {
        use sqlx::Row;
        serde_json::json!({
            "id": row.get::<i64, _>("id"),
            "title": row.get::<String, _>("title"),
            "slug": row.get::<String, _>("slug"),
            "excerpt": row.get::<Option<String>, _>("excerpt"),
            "status": row.get::<String, _>("status"),
            "views": row.get::<i64, _>("views"),
            "author": row.get::<String, _>("author_username"),
        })
    }).collect();
    
    Ok((articles, total))
}
```

---

## Response Metadata

การสร้าง response metadata ที่สมบูรณ์

```rust
use serde::{Deserialize, Serialize};

// Complete response metadata
#[derive(Serialize)]
struct ResponseMeta {
    // Pagination
    pagination: PaginationInfo,
    
    // Query info
    query: QueryInfo,
    
    // Performance
    took_ms: u128,
    
    // Cache
    cached: bool,
    cache_expires_in: Option<u64>,
}

#[derive(Serialize)]
struct PaginationInfo {
    total: i64,
    page: i64,
    per_page: i64,
    total_pages: i64,
    has_next: bool,
    has_prev: bool,
    next_page: Option<i64>,
    prev_page: Option<i64>,
    next_cursor: Option<String>,
    prev_cursor: Option<String>,
}

#[derive(Serialize)]
struct QueryInfo {
    sort_by: String,
    sort_order: String,
    filters: serde_json::Value,
    search: Option<String>,
}

#[derive(Serialize)]
struct CompleteApiResponse<T: Serialize> {
    success: bool,
    data: Vec<T>,
    meta: ResponseMeta,
    errors: Vec<String>,
}

// Handler ที่ return complete response
use actix_web::{web, HttpResponse, Result};
use sqlx::PgPool;
use std::time::Instant;

async fn list_articles_complete(
    pool: web::Data<PgPool>,
    query: web::Query<ArticleQuery>,
) -> Result<HttpResponse> {
    let start = Instant::now();
    let params = query.into_inner();
    
    let page = params.page.unwrap_or(1).max(1);
    let per_page = params.per_page.unwrap_or(20).min(100);
    let sort_by = params.sort_by.clone().unwrap_or_else(|| "created_at".to_string());
    let sort_order = params.sort_order.clone().unwrap_or_else(|| "desc".to_string());
    
    match dynamic_article_query(&pool, &params).await {
        Ok((articles, total)) => {
            let total_pages = (total + per_page - 1) / per_page;
            let duration_ms = start.elapsed().as_millis();
            
            let active_filters = serde_json::json!({
                "search": params.search,
                "author_id": params.author_id,
                "tag": params.tag,
                "status": params.status,
                "min_views": params.min_views,
                "from_date": params.from_date,
                "to_date": params.to_date
            });
            
            Ok(HttpResponse::Ok()
                .append_header(("X-Total-Count", total.to_string()))
                .append_header(("X-Page", page.to_string()))
                .append_header(("X-Per-Page", per_page.to_string()))
                .json(serde_json::json!({
                    "success": true,
                    "data": articles,
                    "meta": {
                        "pagination": {
                            "total": total,
                            "page": page,
                            "per_page": per_page,
                            "total_pages": total_pages,
                            "has_next": page < total_pages,
                            "has_prev": page > 1,
                            "next_page": if page < total_pages { Some(page + 1) } else { None },
                            "prev_page": if page > 1 { Some(page - 1) } else { None }
                        },
                        "query": {
                            "sort_by": sort_by,
                            "sort_order": sort_order,
                            "filters": active_filters,
                            "search": params.search
                        },
                        "took_ms": duration_ms,
                        "cached": false
                    }
                })))
        },
        Err(e) => {
            log::error!("Failed to list articles: {}", e);
            Ok(HttpResponse::InternalServerError().json(serde_json::json!({
                "success": false,
                "error": "Failed to fetch articles"
            })))
        }
    }
}

// App setup
use actix_web::{App, HttpServer, middleware};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));
    
    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = sqlx::PgPool::connect(&database_url).await.expect("Failed to connect to database");
    let pool_data = web::Data::new(pool);
    
    HttpServer::new(move || {
        App::new()
            .app_data(pool_data.clone())
            .wrap(middleware::Logger::default())
            .service(
                web::scope("/api/v1")
                    .route("/users", web::get().to(list_users))
                    .route("/products", web::get().to(list_products_cursor))
                    .route("/articles", web::get().to(list_articles_complete))
                    .route("/search", web::get().to(full_text_search))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Pagination params** - การออกแบบ query parameters สำหรับ pagination
2. **Cursor pagination** - เหมาะกับ real-time data ที่ไม่ต้องการ random access
3. **Offset pagination** - เหมาะกับ data ทั่วไปที่ต้องการ jump to page
4. **Sorting** - การ sort อย่างปลอดภัยด้วย whitelist
5. **Filtering** - Dynamic WHERE clause ด้วย QueryBuilder
6. **Full-text search** - PostgreSQL tsvector/tsquery
7. **Dynamic SQL** - sqlx QueryBuilder สำหรับ complex queries
8. **Response metadata** - total, pages, next, prev, performance info

---

## การนำทาง

- [← Part 029: Form Data and Multipart](../part_029/README.md)
- [→ Part 031: Authentication and JWT](../part_031/README.md)
- [กลับหน้าหลัก](../../README.md)
