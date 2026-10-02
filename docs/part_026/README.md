# Part 026: Error Handling ใน Actix-web

## สารบัญ
- [แนะนำ Error Handling](#แนะนำ-error-handling)
- [ResponseError Trait](#responseerror-trait)
- [Custom Error Types ด้วย thiserror](#custom-error-types-ด้วย-thiserror)
- [Error Mapping](#error-mapping)
- [HTTP Status Codes สำหรับ Errors](#http-status-codes-สำหรับ-errors)
- [Error Middleware](#error-middleware)
- [Validation Errors](#validation-errors)
- [Database Errors → HTTP Errors](#database-errors--http-errors)
- [Error Logging](#error-logging)

---

## แนะนำ Error Handling

การจัดการ error ที่ดีทำให้ API ของเราน่าเชื่อถือและ debug ง่าย ใน Actix-web เราใช้ `ResponseError` trait เพื่อแปลง errors เป็น HTTP responses

### Dependencies

```toml
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
thiserror = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres"] }
log = "0.4"
env_logger = "0.11"
validator = { version = "0.17", features = ["derive"] }
```

---

## ResponseError Trait

`ResponseError` คือ trait ที่ทำให้ error types สามารถแปลงเป็น HTTP response ได้

```rust
use actix_web::{HttpResponse, ResponseError};
use std::fmt;

// Error type พื้นฐาน
#[derive(Debug)]
pub enum ApiError {
    NotFound(String),
    BadRequest(String),
    Unauthorized(String),
    Forbidden(String),
    InternalServerError(String),
    Conflict(String),
    UnprocessableEntity(String),
    TooManyRequests(String),
}

impl fmt::Display for ApiError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ApiError::NotFound(msg) => write!(f, "Not Found: {}", msg),
            ApiError::BadRequest(msg) => write!(f, "Bad Request: {}", msg),
            ApiError::Unauthorized(msg) => write!(f, "Unauthorized: {}", msg),
            ApiError::Forbidden(msg) => write!(f, "Forbidden: {}", msg),
            ApiError::InternalServerError(msg) => write!(f, "Internal Server Error: {}", msg),
            ApiError::Conflict(msg) => write!(f, "Conflict: {}", msg),
            ApiError::UnprocessableEntity(msg) => write!(f, "Unprocessable Entity: {}", msg),
            ApiError::TooManyRequests(msg) => write!(f, "Too Many Requests: {}", msg),
        }
    }
}

// Implement ResponseError เพื่อกำหนด HTTP status code และ response body
impl ResponseError for ApiError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        use actix_web::http::StatusCode;
        match self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::BadRequest(_) => StatusCode::BAD_REQUEST,
            ApiError::Unauthorized(_) => StatusCode::UNAUTHORIZED,
            ApiError::Forbidden(_) => StatusCode::FORBIDDEN,
            ApiError::InternalServerError(_) => StatusCode::INTERNAL_SERVER_ERROR,
            ApiError::Conflict(_) => StatusCode::CONFLICT,
            ApiError::UnprocessableEntity(_) => StatusCode::UNPROCESSABLE_ENTITY,
            ApiError::TooManyRequests(_) => StatusCode::TOO_MANY_REQUESTS,
        }
    }
    
    fn error_response(&self) -> HttpResponse {
        let status = self.status_code();
        let error_code = match self {
            ApiError::NotFound(_) => "not_found",
            ApiError::BadRequest(_) => "bad_request",
            ApiError::Unauthorized(_) => "unauthorized",
            ApiError::Forbidden(_) => "forbidden",
            ApiError::InternalServerError(_) => "internal_server_error",
            ApiError::Conflict(_) => "conflict",
            ApiError::UnprocessableEntity(_) => "unprocessable_entity",
            ApiError::TooManyRequests(_) => "too_many_requests",
        };
        
        HttpResponse::build(status).json(ErrorBody {
            error: error_code.to_string(),
            message: self.to_string(),
            status_code: status.as_u16(),
        })
    }
}

#[derive(serde::Serialize)]
struct ErrorBody {
    error: String,
    message: String,
    status_code: u16,
}

// ตัวอย่างการใช้งาน
use actix_web::{web, Result};

async fn get_user(path: web::Path<u64>) -> Result<HttpResponse, ApiError> {
    let user_id = path.into_inner();
    
    if user_id == 0 {
        return Err(ApiError::BadRequest("User ID must be greater than 0".to_string()));
    }
    
    // Simulate: user not found
    if user_id > 1000 {
        return Err(ApiError::NotFound(format!("User {} not found", user_id)));
    }
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "id": user_id,
        "username": format!("user_{}", user_id)
    })))
}
```

---

## Custom Error Types ด้วย thiserror

`thiserror` ช่วยให้การสร้าง custom error types ง่ายขึ้น

```rust
use thiserror::Error;
use actix_web::{HttpResponse, ResponseError};
use serde::Serialize;
use chrono::Utc;

// Error hierarchy ที่ครอบคลุม
#[derive(Error, Debug)]
pub enum AppError {
    // Authentication errors
    #[error("Authentication required")]
    Unauthenticated,
    
    #[error("Invalid credentials")]
    InvalidCredentials,
    
    #[error("Token expired")]
    TokenExpired,
    
    #[error("Insufficient permissions: {action} on {resource}")]
    InsufficientPermissions {
        action: String,
        resource: String,
    },
    
    // Resource errors
    #[error("{resource} with id '{id}' not found")]
    ResourceNotFound {
        resource: String,
        id: String,
    },
    
    #[error("{resource} already exists")]
    ResourceAlreadyExists {
        resource: String,
    },
    
    // Validation errors
    #[error("Validation failed: {0}")]
    ValidationError(String),
    
    #[error("Invalid input: {field} - {message}")]
    InvalidInput {
        field: String,
        message: String,
    },
    
    // External service errors
    #[error("Database error: {0}")]
    DatabaseError(#[from] sqlx::Error),
    
    #[error("External service error: {service} - {0}")]
    ExternalServiceError {
        service: String,
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },
    
    // Rate limiting
    #[error("Rate limit exceeded. Try again in {retry_after} seconds")]
    RateLimitExceeded {
        retry_after: u64,
    },
    
    // Generic errors
    #[error("Internal server error")]
    InternalError,
    
    #[error("Service temporarily unavailable")]
    ServiceUnavailable,
}

// Structured error response
#[derive(Serialize)]
struct ApiErrorResponse {
    success: bool,
    error: String,
    message: String,
    status_code: u16,
    timestamp: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    details: Option<serde_json::Value>,
    #[serde(skip_serializing_if = "Option::is_none")]
    retry_after: Option<u64>,
}

impl ResponseError for AppError {
    fn status_code(&self) -> actix_web::http::StatusCode {
        use actix_web::http::StatusCode;
        match self {
            AppError::Unauthenticated | AppError::InvalidCredentials | AppError::TokenExpired
                => StatusCode::UNAUTHORIZED,
            AppError::InsufficientPermissions { .. }
                => StatusCode::FORBIDDEN,
            AppError::ResourceNotFound { .. }
                => StatusCode::NOT_FOUND,
            AppError::ResourceAlreadyExists { .. }
                => StatusCode::CONFLICT,
            AppError::ValidationError(_) | AppError::InvalidInput { .. }
                => StatusCode::BAD_REQUEST,
            AppError::DatabaseError(_) | AppError::ExternalServiceError { .. } | AppError::InternalError
                => StatusCode::INTERNAL_SERVER_ERROR,
            AppError::RateLimitExceeded { .. }
                => StatusCode::TOO_MANY_REQUESTS,
            AppError::ServiceUnavailable
                => StatusCode::SERVICE_UNAVAILABLE,
        }
    }
    
    fn error_response(&self) -> HttpResponse {
        let status = self.status_code();
        
        // Log errors ที่ serious
        if status.is_server_error() {
            log::error!("Server error: {:?}", self);
        } else if status == actix_web::http::StatusCode::UNAUTHORIZED 
               || status == actix_web::http::StatusCode::FORBIDDEN {
            log::warn!("Auth error: {}", self);
        }
        
        let error_code = self.error_code();
        let details = self.error_details();
        let retry_after = self.retry_after();
        
        let mut response = HttpResponse::build(status);
        
        // เพิ่ม Retry-After header สำหรับ rate limiting
        if let Some(seconds) = retry_after {
            response.append_header(("Retry-After", seconds.to_string()));
        }
        
        response.json(ApiErrorResponse {
            success: false,
            error: error_code.to_string(),
            message: self.to_string(),
            status_code: status.as_u16(),
            timestamp: Utc::now().to_rfc3339(),
            details,
            retry_after,
        })
    }
}

impl AppError {
    fn error_code(&self) -> &'static str {
        match self {
            AppError::Unauthenticated => "unauthenticated",
            AppError::InvalidCredentials => "invalid_credentials",
            AppError::TokenExpired => "token_expired",
            AppError::InsufficientPermissions { .. } => "insufficient_permissions",
            AppError::ResourceNotFound { .. } => "resource_not_found",
            AppError::ResourceAlreadyExists { .. } => "resource_already_exists",
            AppError::ValidationError(_) => "validation_error",
            AppError::InvalidInput { .. } => "invalid_input",
            AppError::DatabaseError(_) => "database_error",
            AppError::ExternalServiceError { .. } => "external_service_error",
            AppError::RateLimitExceeded { .. } => "rate_limit_exceeded",
            AppError::InternalError => "internal_error",
            AppError::ServiceUnavailable => "service_unavailable",
        }
    }
    
    fn error_details(&self) -> Option<serde_json::Value> {
        match self {
            AppError::InsufficientPermissions { action, resource } => {
                Some(serde_json::json!({
                    "required_action": action,
                    "resource": resource
                }))
            },
            AppError::ResourceNotFound { resource, id } => {
                Some(serde_json::json!({
                    "resource": resource,
                    "id": id
                }))
            },
            _ => None,
        }
    }
    
    fn retry_after(&self) -> Option<u64> {
        match self {
            AppError::RateLimitExceeded { retry_after } => Some(*retry_after),
            _ => None,
        }
    }
}
```

---

## Error Mapping

การแปลง errors จาก external libraries เป็น AppError

```rust
use sqlx::Error as SqlxError;

// Automatic conversion จาก sqlx::Error
// ด้วย #[from] ใน thiserror
// AppError::DatabaseError(#[from] sqlx::Error)
// ทำให้ ? operator ทำงานได้

// Manual mapping สำหรับ error ที่ซับซ้อนกว่า
fn map_sqlx_error(err: SqlxError) -> AppError {
    match err {
        SqlxError::RowNotFound => AppError::ResourceNotFound {
            resource: "Record".to_string(),
            id: "unknown".to_string(),
        },
        SqlxError::Database(db_err) => {
            // ตรวจสอบ error codes
            if db_err.is_unique_violation() {
                AppError::ResourceAlreadyExists {
                    resource: "Record".to_string(),
                }
            } else if db_err.is_foreign_key_violation() {
                AppError::InvalidInput {
                    field: "foreign_key".to_string(),
                    message: "Referenced record does not exist".to_string(),
                }
            } else if db_err.is_not_null_violation() {
                AppError::InvalidInput {
                    field: db_err.column().unwrap_or("unknown").to_string(),
                    message: "Field cannot be null".to_string(),
                }
            } else {
                log::error!("Database error: {:?}", db_err);
                AppError::InternalError
            }
        },
        SqlxError::PoolTimedOut => {
            log::error!("Database pool timeout");
            AppError::ServiceUnavailable
        },
        _ => {
            log::error!("Unexpected database error: {:?}", err);
            AppError::InternalError
        }
    }
}

// Error conversion ด้วย From trait
impl From<std::io::Error> for AppError {
    fn from(err: std::io::Error) -> Self {
        log::error!("IO error: {}", err);
        AppError::InternalError
    }
}

impl From<serde_json::Error> for AppError {
    fn from(err: serde_json::Error) -> Self {
        AppError::InvalidInput {
            field: "body".to_string(),
            message: format!("Invalid JSON: {}", err),
        }
    }
}

// Result type alias
pub type AppResult<T> = Result<T, AppError>;

// ตัวอย่างการใช้งาน ? operator
async fn create_user_handler(
    pool: web::Data<PgPool>,
    body: web::Json<CreateUserDto>,
) -> AppResult<HttpResponse> {
    let dto = body.into_inner();
    
    // Validation
    validate_create_user(&dto)?;
    
    // ตรวจสอบว่า email ซ้ำไหม
    let exists = sqlx::query_scalar!(
        "SELECT EXISTS(SELECT 1 FROM users WHERE email = $1)",
        dto.email
    )
    .fetch_one(pool.get_ref())
    .await
    .map_err(map_sqlx_error)?;
    
    if exists.unwrap_or(false) {
        return Err(AppError::ResourceAlreadyExists {
            resource: "User".to_string(),
        });
    }
    
    // สร้าง user
    let user = sqlx::query_as!(
        UserRecord,
        r#"
        INSERT INTO users (username, email, created_at)
        VALUES ($1, $2, NOW())
        RETURNING id, username, email, created_at
        "#,
        dto.username,
        dto.email
    )
    .fetch_one(pool.get_ref())
    .await
    .map_err(map_sqlx_error)?;
    
    Ok(HttpResponse::Created().json(user))
}

#[derive(serde::Deserialize)]
struct CreateUserDto {
    username: String,
    email: String,
}

fn validate_create_user(dto: &CreateUserDto) -> AppResult<()> {
    if dto.username.trim().is_empty() {
        return Err(AppError::InvalidInput {
            field: "username".to_string(),
            message: "Username cannot be empty".to_string(),
        });
    }
    if !dto.email.contains('@') {
        return Err(AppError::InvalidInput {
            field: "email".to_string(),
            message: "Invalid email format".to_string(),
        });
    }
    Ok(())
}
```

---

## HTTP Status Codes สำหรับ Errors

แนวทางการใช้ HTTP status codes อย่างถูกต้อง

```rust
use actix_web::http::StatusCode;

// 4xx Client Errors - client ส่ง request ผิด
// 400 Bad Request - request syntax ผิด, invalid parameters
// 401 Unauthorized - ต้อง authenticate ก่อน
// 403 Forbidden - authenticate แล้วแต่ไม่มีสิทธิ์
// 404 Not Found - resource ไม่มี
// 405 Method Not Allowed - method ไม่ถูก
// 409 Conflict - conflict กับ state ปัจจุบัน (เช่น duplicate)
// 410 Gone - resource เคยมีแต่ถูกลบแล้ว
// 422 Unprocessable Entity - syntax ถูกแต่ semantic ผิด (validation error)
// 429 Too Many Requests - rate limited

// 5xx Server Errors - server เกิด error
// 500 Internal Server Error - general server error
// 502 Bad Gateway - upstream server error
// 503 Service Unavailable - server overloaded หรือ maintenance
// 504 Gateway Timeout - upstream timeout

// ตัวอย่าง error responses ที่ถูกต้อง
fn error_examples() {
    // 400: ข้อมูลไม่ครบหรือ format ผิด
    let bad_request = HttpResponse::BadRequest().json(serde_json::json!({
        "error": "bad_request",
        "message": "Invalid request format",
        "details": {
            "field": "email",
            "message": "Email format is invalid"
        }
    }));
    
    // 401: ไม่มี token หรือ token ผิด
    let unauthorized = HttpResponse::Unauthorized()
        .append_header(("WWW-Authenticate", r#"Bearer realm="api", error="invalid_token""#))
        .json(serde_json::json!({
            "error": "unauthorized",
            "message": "Authentication required"
        }));
    
    // 403: มี token แต่ไม่มีสิทธิ์
    let forbidden = HttpResponse::Forbidden().json(serde_json::json!({
        "error": "forbidden",
        "message": "You don't have permission to perform this action",
        "required_role": "admin"
    }));
    
    // 404: ไม่เจอ resource
    let not_found = HttpResponse::NotFound().json(serde_json::json!({
        "error": "not_found",
        "message": "User with id '123' not found"
    }));
    
    // 409: conflict
    let conflict = HttpResponse::Conflict().json(serde_json::json!({
        "error": "conflict",
        "message": "User with this email already exists"
    }));
    
    // 422: validation failed
    let unprocessable = HttpResponse::UnprocessableEntity().json(serde_json::json!({
        "error": "validation_failed",
        "message": "Request validation failed",
        "errors": [
            {"field": "age", "message": "Must be at least 18"},
            {"field": "password", "message": "Must be at least 8 characters"}
        ]
    }));
    
    // 429: rate limited
    let too_many = HttpResponse::TooManyRequests()
        .append_header(("Retry-After", "60"))
        .append_header(("X-RateLimit-Limit", "100"))
        .append_header(("X-RateLimit-Remaining", "0"))
        .json(serde_json::json!({
            "error": "rate_limit_exceeded",
            "message": "Too many requests",
            "retry_after": 60
        }));
    
    // 500: internal error (ไม่เปิดเผย details)
    let internal = HttpResponse::InternalServerError().json(serde_json::json!({
        "error": "internal_server_error",
        "message": "An unexpected error occurred"
    }));
    
    // 503: service unavailable
    let unavailable = HttpResponse::ServiceUnavailable()
        .append_header(("Retry-After", "30"))
        .json(serde_json::json!({
            "error": "service_unavailable",
            "message": "Service is temporarily unavailable for maintenance"
        }));
}
```

---

## Error Middleware

Middleware สำหรับ centralized error handling

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
    body::EitherBody,
};
use futures_util::future::LocalBoxFuture;
use std::future::{ready, Ready};
use std::rc::Rc;

pub struct ErrorHandlerMiddleware {
    include_stack_trace: bool,
}

impl ErrorHandlerMiddleware {
    pub fn new() -> Self {
        ErrorHandlerMiddleware {
            include_stack_trace: std::env::var("DEBUG")
                .map(|v| v == "true")
                .unwrap_or(false),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for ErrorHandlerMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<EitherBody<B>>;
    type Error = Error;
    type InitError = ();
    type Transform = ErrorHandlerService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(ErrorHandlerService {
            service: Rc::new(service),
            include_stack_trace: self.include_stack_trace,
        }))
    }
}

pub struct ErrorHandlerService<S> {
    service: Rc<S>,
    include_stack_trace: bool,
}

impl<S, B> Service<ServiceRequest> for ErrorHandlerService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<EitherBody<B>>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let include_stack_trace = self.include_stack_trace;
        let request_id = req.headers()
            .get("X-Request-ID")
            .and_then(|v| v.to_str().ok())
            .map(String::from)
            .unwrap_or_else(|| uuid::Uuid::new_v4().to_string());
        let method = req.method().to_string();
        let path = req.path().to_string();
        
        Box::pin(async move {
            match service.call(req).await {
                Ok(res) => {
                    let status = res.status();
                    
                    // Log all 4xx and 5xx
                    if status.is_client_error() || status.is_server_error() {
                        log::warn!(
                            "[{}] {} {} -> {} {}",
                            request_id,
                            method,
                            path,
                            status.as_u16(),
                            status.canonical_reason().unwrap_or("Unknown")
                        );
                    }
                    
                    Ok(res.map_into_left_body())
                },
                Err(err) => {
                    log::error!("[{}] Unhandled error: {:?}", request_id, err);
                    Err(err)
                }
            }
        })
    }
}

// Custom error handlers สำหรับ specific error types
use actix_web::middleware::ErrorHandlers;
use actix_web::dev::ServiceResponse;

fn not_found_handler<B>(
    res: ServiceResponse<B>,
) -> actix_web::Result<actix_web::middleware::ErrorHandlerResponse<B>> {
    let (req, _res) = res.into_parts();
    
    let response = HttpResponse::NotFound()
        .json(serde_json::json!({
            "error": "not_found",
            "message": format!("Path '{}' not found", req.path()),
            "timestamp": chrono::Utc::now().to_rfc3339()
        }));
    
    let res = ServiceResponse::new(req, response)
        .map_into_right_body();
    
    Ok(actix_web::middleware::ErrorHandlerResponse::Response(res))
}

// App setup ด้วย error handlers
use actix_web::http::StatusCode;

fn create_app_with_error_handlers() -> actix_web::App<
    impl actix_web::dev::ServiceFactory<
        ServiceRequest,
        Config = (),
        Response = ServiceResponse,
        Error = Error,
        InitError = (),
    >
> {
    App::new()
        .wrap(
            ErrorHandlers::new()
                .handler(StatusCode::NOT_FOUND, not_found_handler)
        )
}
```

---

## Validation Errors

การจัดการ validation errors อย่างละเอียด

```rust
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

// Validation error structure
#[derive(Debug, Serialize)]
pub struct ValidationErrors {
    pub errors: HashMap<String, Vec<String>>,
}

impl ValidationErrors {
    pub fn new() -> Self {
        ValidationErrors {
            errors: HashMap::new(),
        }
    }
    
    pub fn add(&mut self, field: &str, message: &str) {
        self.errors
            .entry(field.to_string())
            .or_default()
            .push(message.to_string());
    }
    
    pub fn is_empty(&self) -> bool {
        self.errors.is_empty()
    }
}

// Manual validation
#[derive(Debug, Deserialize)]
struct RegisterRequest {
    username: String,
    email: String,
    password: String,
    password_confirm: String,
    age: u32,
    agree_to_terms: bool,
}

fn validate_register(req: &RegisterRequest) -> Result<(), ValidationErrors> {
    let mut errors = ValidationErrors::new();
    
    // Username validation
    if req.username.trim().is_empty() {
        errors.add("username", "Username is required");
    } else if req.username.len() < 3 {
        errors.add("username", "Username must be at least 3 characters");
    } else if req.username.len() > 50 {
        errors.add("username", "Username must be 50 characters or less");
    } else if !req.username.chars().all(|c| c.is_alphanumeric() || c == '_') {
        errors.add("username", "Username can only contain letters, numbers, and underscores");
    }
    
    // Email validation
    if req.email.trim().is_empty() {
        errors.add("email", "Email is required");
    } else if !is_valid_email(&req.email) {
        errors.add("email", "Invalid email format");
    }
    
    // Password validation
    if req.password.len() < 8 {
        errors.add("password", "Password must be at least 8 characters");
    }
    if !req.password.chars().any(|c| c.is_uppercase()) {
        errors.add("password", "Password must contain at least one uppercase letter");
    }
    if !req.password.chars().any(|c| c.is_numeric()) {
        errors.add("password", "Password must contain at least one number");
    }
    if !req.password.chars().any(|c| "!@#$%^&*()_+-=[]{}|;':\",./<>?".contains(c)) {
        errors.add("password", "Password must contain at least one special character");
    }
    
    // Password confirm
    if req.password != req.password_confirm {
        errors.add("password_confirm", "Passwords do not match");
    }
    
    // Age validation
    if req.age < 18 {
        errors.add("age", "Must be at least 18 years old");
    }
    if req.age > 120 {
        errors.add("age", "Invalid age");
    }
    
    // Terms
    if !req.agree_to_terms {
        errors.add("agree_to_terms", "You must agree to the terms and conditions");
    }
    
    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}

fn is_valid_email(email: &str) -> bool {
    let parts: Vec<&str> = email.splitn(2, '@').collect();
    if parts.len() != 2 {
        return false;
    }
    let local = parts[0];
    let domain = parts[1];
    !local.is_empty() && domain.contains('.') && !domain.starts_with('.')
}

async fn register_handler(
    body: web::Json<RegisterRequest>,
) -> HttpResponse {
    let req = body.into_inner();
    
    match validate_register(&req) {
        Ok(()) => {
            // ดำเนินการสร้าง user
            HttpResponse::Created().json(serde_json::json!({
                "message": "User registered successfully",
                "username": req.username
            }))
        },
        Err(validation_errors) => {
            HttpResponse::UnprocessableEntity().json(serde_json::json!({
                "error": "validation_failed",
                "message": "Request validation failed",
                "errors": validation_errors.errors,
                "error_count": validation_errors.errors.len()
            }))
        }
    }
}
```

---

## Database Errors → HTTP Errors

การแปลง database errors เป็น HTTP errors ที่เหมาะสม

```rust
use sqlx::Error as SqlxError;
use actix_web::HttpResponse;

// Comprehensive database error mapping
fn handle_db_error(err: SqlxError, context: &str) -> HttpResponse {
    match err {
        SqlxError::RowNotFound => {
            HttpResponse::NotFound().json(serde_json::json!({
                "error": "not_found",
                "message": format!("{} not found", context)
            }))
        },
        SqlxError::Database(ref db_err) => {
            // PostgreSQL specific error codes
            let pg_code = db_err.code().map(|c| c.to_string());
            
            match pg_code.as_deref() {
                // 23505: unique_violation
                Some("23505") => {
                    let constraint = db_err.constraint().unwrap_or("unknown");
                    let field = constraint.split('_').last().unwrap_or("field");
                    
                    HttpResponse::Conflict().json(serde_json::json!({
                        "error": "duplicate_entry",
                        "message": format!("{} already exists", context),
                        "field": field
                    }))
                },
                // 23503: foreign_key_violation
                Some("23503") => {
                    HttpResponse::BadRequest().json(serde_json::json!({
                        "error": "invalid_reference",
                        "message": "Referenced resource does not exist"
                    }))
                },
                // 23502: not_null_violation
                Some("23502") => {
                    let column = db_err.column().unwrap_or("unknown");
                    HttpResponse::BadRequest().json(serde_json::json!({
                        "error": "null_violation",
                        "message": format!("Field '{}' cannot be null", column)
                    }))
                },
                // 23514: check_violation
                Some("23514") => {
                    HttpResponse::BadRequest().json(serde_json::json!({
                        "error": "constraint_violation",
                        "message": "Data violates database constraints"
                    }))
                },
                // 42P01: undefined_table
                Some("42P01") => {
                    log::error!("Table not found error: {:?}", db_err);
                    HttpResponse::InternalServerError().json(serde_json::json!({
                        "error": "internal_error",
                        "message": "Internal server error"
                    }))
                },
                _ => {
                    log::error!("Database error [{}]: {:?}", context, db_err);
                    HttpResponse::InternalServerError().json(serde_json::json!({
                        "error": "database_error",
                        "message": "A database error occurred"
                    }))
                }
            }
        },
        SqlxError::PoolTimedOut => {
            log::error!("Database pool timeout");
            HttpResponse::ServiceUnavailable()
                .append_header(("Retry-After", "5"))
                .json(serde_json::json!({
                    "error": "service_unavailable",
                    "message": "Database is temporarily unavailable"
                }))
        },
        SqlxError::PoolClosed => {
            log::error!("Database pool closed");
            HttpResponse::ServiceUnavailable().json(serde_json::json!({
                "error": "service_unavailable",
                "message": "Database connection is unavailable"
            }))
        },
        _ => {
            log::error!("Unexpected database error [{}]: {:?}", context, err);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "internal_error",
                "message": "An unexpected error occurred"
            }))
        }
    }
}
```

---

## Error Logging

การ log errors อย่างเป็นระบบ

```rust
use log::{error, warn, info, debug};
use std::fmt;

// Structured logging สำหรับ errors
struct ErrorLog {
    request_id: Option<String>,
    method: String,
    path: String,
    status_code: u16,
    error: String,
    user_id: Option<u64>,
    duration_ms: u128,
}

impl ErrorLog {
    fn log(&self) {
        let user_str = self.user_id
            .map(|id| format!(" user_id={}", id))
            .unwrap_or_default();
        
        let req_id = self.request_id.as_deref().unwrap_or("unknown");
        
        if self.status_code >= 500 {
            error!(
                "[{}] {} {} {} ({}ms){} - {}",
                req_id,
                self.method,
                self.path,
                self.status_code,
                self.duration_ms,
                user_str,
                self.error
            );
        } else if self.status_code >= 400 {
            warn!(
                "[{}] {} {} {} ({}ms){} - {}",
                req_id,
                self.method,
                self.path,
                self.status_code,
                self.duration_ms,
                user_str,
                self.error
            );
        }
    }
}

// Error context สำหรับเพิ่มข้อมูลให้ errors
trait ErrorContext<T, E> {
    fn with_context(self, context: &str) -> Result<T, String>;
}

impl<T, E: fmt::Display> ErrorContext<T, E> for Result<T, E> {
    fn with_context(self, context: &str) -> Result<T, String> {
        self.map_err(|e| format!("{}: {}", context, e))
    }
}

// ตัวอย่าง handler พร้อม error logging
async fn handler_with_logging(
    req: actix_web::HttpRequest,
    pool: web::Data<PgPool>,
    path: web::Path<u64>,
) -> HttpResponse {
    let start = std::time::Instant::now();
    let request_id = req.headers()
        .get("X-Request-ID")
        .and_then(|v| v.to_str().ok())
        .map(String::from);
    let method = req.method().to_string();
    let request_path = req.path().to_string();
    
    let user_id = path.into_inner();
    
    let result = sqlx::query!(
        "SELECT id, username FROM users WHERE id = $1",
        user_id as i64
    )
    .fetch_optional(pool.get_ref())
    .await;
    
    let duration_ms = start.elapsed().as_millis();
    
    match result {
        Ok(Some(user)) => {
            debug!("[{:?}] User {} fetched in {}ms",
                request_id, user_id, duration_ms);
            HttpResponse::Ok().json(serde_json::json!({
                "id": user.id,
                "username": user.username
            }))
        },
        Ok(None) => {
            ErrorLog {
                request_id,
                method,
                path: request_path,
                status_code: 404,
                error: format!("User {} not found", user_id),
                user_id: None,
                duration_ms,
            }.log();
            
            HttpResponse::NotFound().json(serde_json::json!({
                "error": "not_found",
                "message": format!("User {} not found", user_id)
            }))
        },
        Err(e) => {
            ErrorLog {
                request_id,
                method,
                path: request_path,
                status_code: 500,
                error: format!("Database error: {}", e),
                user_id: None,
                duration_ms,
            }.log();
            
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "internal_error",
                "message": "Failed to fetch user"
            }))
        }
    }
}
```

---

## ตัวอย่างสมบูรณ์: Error Handling System

```rust
use actix_web::{App, HttpServer, web, middleware};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(
        env_logger::Env::default().default_filter_or("info")
    );
    
    HttpServer::new(|| {
        App::new()
            .wrap(middleware::Logger::default())
            .wrap(ErrorHandlerMiddleware::new())
            .service(
                web::scope("/api/v1")
                    .route("/register", web::post().to(register_handler))
                    .route("/users/{id}", web::get().to(handler_with_logging))
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

1. **ResponseError trait** - แปลง error เป็น HTTP response
2. **thiserror** - สร้าง custom error types อย่างสะดวก
3. **Error mapping** - แปลง errors จาก external libraries
4. **HTTP status codes** - การใช้ status codes อย่างถูกต้อง
5. **Error middleware** - Centralized error handling
6. **Validation errors** - การตรวจสอบ input อย่างละเอียด
7. **Database errors** - แปลง DB errors เป็น HTTP errors
8. **Error logging** - การ log errors อย่างเป็นระบบ

---

## การนำทาง

- [← Part 025: State Management](../part_025/README.md)
- [→ Part 027: Static Files and Templates](../part_027/README.md)
- [กลับหน้าหลัก](../../README.md)
