# Part 022: Routes, Handlers, และ Extractors 🛣️

## 🎯 เป้าหมายของ Part นี้

- Route macros ทุกประเภท
- Extractors: Path, Query, Json, Form
- Custom Extractors
- Request validation
- Response building

---

## 1. Route Macros

### 1.1 HTTP Methods

```rust
use actix_web::{get, post, put, patch, delete, head, options, web, HttpResponse};

// GET
#[get("/users")]
async fn get_users() -> HttpResponse {
    HttpResponse::Ok().body("GET users")
}

// POST
#[post("/users")]
async fn create_user() -> HttpResponse {
    HttpResponse::Created().body("Created user")
}

// PUT (full update)
#[put("/users/{id}")]
async fn update_user(path: web::Path<u32>) -> HttpResponse {
    HttpResponse::Ok().body(format!("Updated user {}", path))
}

// PATCH (partial update)
#[patch("/users/{id}")]
async fn patch_user(path: web::Path<u32>) -> HttpResponse {
    HttpResponse::Ok().body(format!("Patched user {}", path))
}

// DELETE
#[delete("/users/{id}")]
async fn delete_user(path: web::Path<u32>) -> HttpResponse {
    HttpResponse::NoContent().finish()
}

// HEAD (ไม่มี body)
#[head("/users")]
async fn head_users() -> HttpResponse {
    HttpResponse::Ok()
        .insert_header(("X-Total-Count", "100"))
        .finish()
}

// OPTIONS (CORS preflight)
#[options("/users")]
async fn options_users() -> HttpResponse {
    HttpResponse::Ok()
        .insert_header(("Allow", "GET, POST, PUT, DELETE"))
        .finish()
}

// web::route() สำหรับ custom methods
async fn any_method() -> HttpResponse {
    HttpResponse::Ok().body("Any method")
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::HttpServer;

    HttpServer::new(|| {
        actix_web::App::new()
            .service(get_users)
            .service(create_user)
            .service(update_user)
            .service(patch_user)
            .service(delete_user)
            .service(head_users)
            .service(options_users)
            .route("/any", web::route().to(any_method))
            .route("/get-or-post", web::get().to(any_method))
            // Multiple methods on same route
            .route("/multi", web::get().to(any_method))
            .route("/multi", web::post().to(any_method))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 2. Extractors

### 2.1 Path Extractor

```rust
use actix_web::{get, web, HttpResponse};
use serde::Deserialize;

// Single path parameter
#[get("/users/{id}")]
async fn get_user(path: web::Path<u32>) -> HttpResponse {
    let id = path.into_inner();
    HttpResponse::Ok().body(format!("User ID: {}", id))
}

// Multiple path parameters (tuple)
#[get("/users/{user_id}/posts/{post_id}")]
async fn get_user_post(path: web::Path<(u32, u32)>) -> HttpResponse {
    let (user_id, post_id) = path.into_inner();
    HttpResponse::Ok().body(format!("User {} Post {}", user_id, post_id))
}

// Struct path parameters
#[derive(Deserialize)]
struct UserPostPath {
    user_id: u32,
    post_id: u32,
}

#[get("/users/{user_id}/comments/{post_id}")]
async fn get_user_comment(path: web::Path<UserPostPath>) -> HttpResponse {
    let path = path.into_inner();
    HttpResponse::Ok().body(format!(
        "User {} Comment on Post {}",
        path.user_id, path.post_id
    ))
}

// Catch-all (tail)
#[get("/files/{filename:.*}")]
async fn get_file(path: web::Path<String>) -> HttpResponse {
    let filename = path.into_inner();
    HttpResponse::Ok().body(format!("File: {}", filename))
}
```

### 2.2 Query String Extractor

```rust
use actix_web::{get, web, HttpResponse};
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct PaginationQuery {
    page: Option<u32>,
    per_page: Option<u32>,
    sort_by: Option<String>,
    order: Option<String>,
    search: Option<String>,
}

impl Default for PaginationQuery {
    fn default() -> Self {
        Self {
            page: Some(1),
            per_page: Some(20),
            sort_by: Some("created_at".to_string()),
            order: Some("desc".to_string()),
            search: None,
        }
    }
}

#[get("/users")]
async fn list_users(query: web::Query<PaginationQuery>) -> HttpResponse {
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(20).min(100);  // max 100

    // แสดงผลสำหรับตัวอย่าง
    use serde::Serialize;
    #[derive(Serialize)]
    struct UsersResponse {
        page: u32,
        per_page: u32,
        total: u32,
        users: Vec<String>,
    }

    HttpResponse::Ok().json(UsersResponse {
        page,
        per_page,
        total: 100,
        users: vec![
            format!("User_{}", page * per_page - per_page + 1),
            format!("User_{}", page * per_page),
        ],
    })
}

// Multi-value query params
#[derive(Deserialize, Debug)]
struct FilterQuery {
    #[serde(rename = "tag")]
    tags: Option<Vec<String>>,
    status: Option<Vec<String>>,
}

#[get("/posts")]
async fn list_posts(query: web::Query<FilterQuery>) -> HttpResponse {
    println!("Tags: {:?}", query.tags);
    println!("Status: {:?}", query.status);
    HttpResponse::Ok().body("Posts filtered")
}
```

### 2.3 JSON Body Extractor

```rust
use actix_web::{post, put, web, HttpResponse};
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Serialize, Debug)]
struct CreateUserRequest {
    username: String,
    email: String,
    password: String,
    age: Option<u32>,
    role: Option<String>,
}

#[derive(Deserialize, Serialize, Debug)]
struct UpdateUserRequest {
    username: Option<String>,
    email: Option<String>,
    age: Option<u32>,
}

#[derive(Serialize)]
struct UserResponse {
    id: u32,
    username: String,
    email: String,
    age: Option<u32>,
    role: String,
    created_at: String,
}

#[post("/users")]
async fn create_user(body: web::Json<CreateUserRequest>) -> HttpResponse {
    println!("Creating user: {:?}", body);

    // Simulate user creation
    let user = UserResponse {
        id: 1,
        username: body.username.clone(),
        email: body.email.clone(),
        age: body.age,
        role: body.role.clone().unwrap_or_else(|| "user".to_string()),
        created_at: "2024-01-01T00:00:00Z".to_string(),
    };

    HttpResponse::Created().json(user)
}

// Custom JSON config (ขนาด body สูงสุด)
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    use actix_web::{HttpServer, App};

    HttpServer::new(|| {
        // กำหนด JSON config
        let json_cfg = web::JsonConfig::default()
            .limit(1024 * 1024)  // 1 MB max
            .error_handler(|err, _req| {
                let response = HttpResponse::BadRequest().json(
                    serde_json::json!({
                        "error": "Invalid JSON",
                        "message": err.to_string()
                    })
                );
                actix_web::error::InternalError::from_response(err, response).into()
            });

        App::new()
            .app_data(json_cfg)
            .service(create_user)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### 2.4 Form Extractor

```rust
use actix_web::{post, web, HttpResponse};
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct LoginForm {
    username: String,
    password: String,
    remember_me: Option<bool>,
}

#[post("/login")]
async fn login(form: web::Form<LoginForm>) -> HttpResponse {
    println!("Login attempt: {}", form.username);

    // ตรวจสอบ credentials (ตัวอย่าง)
    if form.username == "admin" && form.password == "password" {
        HttpResponse::Ok().json(serde_json::json!({
            "success": true,
            "token": "fake_jwt_token",
            "remember_me": form.remember_me.unwrap_or(false)
        }))
    } else {
        HttpResponse::Unauthorized().json(serde_json::json!({
            "success": false,
            "error": "Invalid credentials"
        }))
    }
}
```

### 2.5 Header Extractor

```rust
use actix_web::{get, HttpRequest, HttpResponse};
use actix_web::http::header;

#[get("/protected")]
async fn protected(req: HttpRequest) -> HttpResponse {
    // Get specific header
    let auth = req.headers().get(header::AUTHORIZATION);

    match auth {
        Some(token) => {
            let token = token.to_str().unwrap_or("");
            if token.starts_with("Bearer ") {
                let jwt = &token[7..];
                HttpResponse::Ok().json(serde_json::json!({
                    "message": "Authorized",
                    "token_preview": &jwt[..10.min(jwt.len())]
                }))
            } else {
                HttpResponse::Unauthorized().json(serde_json::json!({
                    "error": "Invalid token format"
                }))
            }
        },
        None => HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Missing Authorization header"
        }))
    }
}

// ด้วย custom extractor
use actix_web::{FromRequest, Error};
use actix_web::dev::Payload;
use futures::future::{ready, Ready};

struct BearerToken(String);

impl FromRequest for BearerToken {
    type Error = Error;
    type Future = Ready<Result<Self, Self::Error>>;

    fn from_request(req: &HttpRequest, _: &mut Payload) -> Self::Future {
        let token = req.headers()
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|s| s.strip_prefix("Bearer "))
            .map(|s| s.to_string());

        match token {
            Some(t) => ready(Ok(BearerToken(t))),
            None => ready(Err(actix_web::error::ErrorUnauthorized("Missing token"))),
        }
    }
}

#[get("/api/data")]
async fn get_data(token: BearerToken) -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "data": "secret data",
        "token_used": &token.0[..5]
    }))
}
```

---

## 3. Request Validation

### 3.1 Manual Validation

```rust
use actix_web::{post, web, HttpResponse};
use serde::{Deserialize, Serialize};

#[derive(Deserialize, Debug)]
struct CreateProductRequest {
    name: String,
    price: f64,
    stock: u32,
    category: String,
    description: Option<String>,
}

#[derive(Serialize)]
struct ValidationError {
    field: String,
    message: String,
}

fn validate_create_product(req: &CreateProductRequest) -> Vec<ValidationError> {
    let mut errors = Vec::new();

    if req.name.trim().is_empty() {
        errors.push(ValidationError {
            field: "name".to_string(),
            message: "Name is required".to_string(),
        });
    } else if req.name.len() > 100 {
        errors.push(ValidationError {
            field: "name".to_string(),
            message: "Name must be at most 100 characters".to_string(),
        });
    }

    if req.price < 0.0 {
        errors.push(ValidationError {
            field: "price".to_string(),
            message: "Price must be non-negative".to_string(),
        });
    }

    let valid_categories = ["electronics", "clothing", "food", "books"];
    if !valid_categories.contains(&req.category.to_lowercase().as_str()) {
        errors.push(ValidationError {
            field: "category".to_string(),
            message: format!("Category must be one of: {:?}", valid_categories),
        });
    }

    errors
}

#[post("/products")]
async fn create_product(body: web::Json<CreateProductRequest>) -> HttpResponse {
    let errors = validate_create_product(&body);

    if !errors.is_empty() {
        return HttpResponse::UnprocessableEntity().json(serde_json::json!({
            "success": false,
            "errors": errors
        }));
    }

    // Process valid request
    HttpResponse::Created().json(serde_json::json!({
        "success": true,
        "product": {
            "id": 1,
            "name": body.name,
            "price": body.price
        }
    }))
}
```

### 3.2 Validator Crate

```toml
# Cargo.toml
validator = { version = "0.18", features = ["derive"] }
```

```rust
use actix_web::{post, web, HttpResponse};
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Deserialize, Serialize, Validate, Debug)]
struct RegisterRequest {
    #[validate(length(min = 3, max = 50, message = "Username must be 3-50 chars"))]
    #[validate(regex(path = "USERNAME_REGEX", message = "Username can only contain alphanumeric and underscore"))]
    username: String,

    #[validate(email(message = "Invalid email address"))]
    email: String,

    #[validate(length(min = 8, message = "Password must be at least 8 characters"))]
    #[validate(regex(path = "PASSWORD_REGEX", message = "Password must contain uppercase, lowercase, and digit"))]
    password: String,

    #[validate(must_match(other = "password", message = "Passwords do not match"))]
    confirm_password: String,

    #[validate(range(min = 13, max = 120, message = "Age must be between 13 and 120"))]
    age: u32,

    #[validate(url(message = "Invalid URL"))]
    website: Option<String>,
}

use once_cell::sync::Lazy;
use regex::Regex;

static USERNAME_REGEX: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^[a-zA-Z0-9_]+$").unwrap()
});

static PASSWORD_REGEX: Lazy<Regex> = Lazy::new(|| {
    Regex::new(r"^(?=.*[A-Z])(?=.*[a-z])(?=.*\d).+$").unwrap()
});

#[post("/register")]
async fn register(body: web::Json<RegisterRequest>) -> HttpResponse {
    match body.validate() {
        Ok(()) => {
            // Process valid registration
            HttpResponse::Created().json(serde_json::json!({
                "success": true,
                "message": "Registration successful",
                "username": body.username
            }))
        },
        Err(errors) => {
            HttpResponse::UnprocessableEntity().json(serde_json::json!({
                "success": false,
                "errors": errors
            }))
        }
    }
}
```

---

## 4. สรุปและ Exercises

### 4.1 สิ่งที่เรียนรู้

✅ Route macros (get, post, put, patch, delete)  
✅ Path Extractor  
✅ Query String Extractor  
✅ JSON Body Extractor  
✅ Form Extractor  
✅ Header Extractor  
✅ Custom Extractors  
✅ Input Validation  

### 4.2 Exercises

**Exercise: Blog API**
```
GET  /api/posts?page=1&per_page=10&tag=rust
GET  /api/posts/{id}
POST /api/posts  (body: {title, content, tags[]})
PUT  /api/posts/{id}
DELETE /api/posts/{id}
GET  /api/posts/{id}/comments
POST /api/posts/{id}/comments
```

---

*[← Part 021: Actix-web Hello World](../part_021/README.md) | [Part 023: JSON Request/Response →](../part_023/README.md)*
