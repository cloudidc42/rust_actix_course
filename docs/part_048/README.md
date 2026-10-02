# Part 048: Input Validation Advanced ✅

## 🎯 เป้าหมายของ Part นี้

- ใช้ `validator` crate แบบ advanced
- Custom validators
- Nested struct validation
- Cross-field validation
- Async database uniqueness check
- Sanitization (HTML, SQL)
- File upload validation
- Complete user registration validation

---

## 1. Setup

### 1.1 Cargo.toml

```toml
[package]
name = "input-validation"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-multipart = "0.6"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
validator = { version = "0.16", features = ["derive", "phone"] }
regex = "1"
sanitize-filename = "0.5"
html-escape = "0.2"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
futures = "0.3"
mime = "0.3"
bytes = "1"
```

---

## 2. Basic Validator Usage

```rust
// src/basic_validation.rs
use serde::{Deserialize, Serialize};
use validator::{Validate, ValidationError};
use regex::Regex;

#[derive(Debug, Deserialize, Validate, Serialize)]
pub struct CreateUserRequest {
    #[validate(
        length(min = 3, max = 50, message = "Username must be 3-50 characters"),
        regex(path = "USERNAME_REGEX", message = "Username can only contain letters, numbers, and underscores")
    )]
    pub username: String,
    
    #[validate(
        email(message = "Invalid email format"),
        length(max = 255, message = "Email too long")
    )]
    pub email: String,
    
    #[validate(
        length(min = 8, max = 128, message = "Password must be 8-128 characters"),
        custom = "validate_password_strength"
    )]
    pub password: String,
    
    #[validate(must_match(other = "password", message = "Passwords do not match"))]
    pub confirm_password: String,
    
    #[validate(range(min = 13, max = 120, message = "Age must be between 13 and 120"))]
    pub age: Option<u8>,
    
    #[validate(url(message = "Invalid website URL"))]
    pub website: Option<String>,
    
    #[validate(phone(message = "Invalid phone number"))]
    pub phone: Option<String>,
}

// Regex สำหรับ username
lazy_static::lazy_static! {
    static ref USERNAME_REGEX: Regex = Regex::new(r"^[a-zA-Z0-9_]+$").unwrap();
}

/// Custom validator สำหรับ password strength
fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_uppercase = password.chars().any(|c| c.is_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_ascii_digit());
    
    if !has_uppercase || !has_lowercase || !has_digit {
        let mut error = ValidationError::new("password_strength");
        error.message = Some(std::borrow::Cow::Borrowed(
            "Password must contain uppercase, lowercase, and digits"
        ));
        return Err(error);
    }
    
    Ok(())
}
```

---

## 3. Custom Validators

```rust
// src/validators/custom.rs
use validator::ValidationError;
use regex::Regex;

lazy_static::lazy_static! {
    static ref THAI_PHONE: Regex = Regex::new(r"^(\+66|0)[0-9]{8,9}$").unwrap();
    static ref SLUG_REGEX: Regex = Regex::new(r"^[a-z0-9]+(?:-[a-z0-9]+)*$").unwrap();
    static ref HEX_COLOR: Regex = Regex::new(r"^#[0-9A-Fa-f]{6}$").unwrap();
    static ref USERNAME_NO_RESERVED: Regex = Regex::new(r"^[a-zA-Z0-9_]+$").unwrap();
}

/// Validate เบอร์โทรศัพท์ไทย
pub fn validate_thai_phone(phone: &str) -> Result<(), ValidationError> {
    if !THAI_PHONE.is_match(phone) {
        let mut err = ValidationError::new("invalid_thai_phone");
        err.message = Some(std::borrow::Cow::Borrowed("Invalid Thai phone number format"));
        return Err(err);
    }
    Ok(())
}

/// Validate URL slug
pub fn validate_slug(slug: &str) -> Result<(), ValidationError> {
    if !SLUG_REGEX.is_match(slug) {
        let mut err = ValidationError::new("invalid_slug");
        err.message = Some(std::borrow::Cow::Borrowed(
            "Slug must be lowercase letters, numbers, and hyphens only"
        ));
        return Err(err);
    }
    Ok(())
}

/// Validate hex color code
pub fn validate_hex_color(color: &str) -> Result<(), ValidationError> {
    if !HEX_COLOR.is_match(color) {
        let mut err = ValidationError::new("invalid_hex_color");
        err.message = Some(std::borrow::Cow::Borrowed("Invalid hex color format (e.g., #FF0000)"));
        return Err(err);
    }
    Ok(())
}

/// Validate username ไม่ใช้ reserved words
pub fn validate_username_not_reserved(username: &str) -> Result<(), ValidationError> {
    let reserved = [
        "admin", "root", "system", "null", "undefined", "true", "false",
        "api", "www", "mail", "ftp", "smtp", "support", "help",
    ];
    
    let lower = username.to_lowercase();
    if reserved.contains(&lower.as_str()) {
        let mut err = ValidationError::new("reserved_username");
        err.message = Some(std::borrow::Cow::Borrowed("This username is reserved"));
        return Err(err);
    }
    
    if !USERNAME_NO_RESERVED.is_match(username) {
        let mut err = ValidationError::new("invalid_username");
        err.message = Some(std::borrow::Cow::Borrowed("Username contains invalid characters"));
        return Err(err);
    }
    
    Ok(())
}

/// Validate ว่าไม่มี profanity (ตัวอย่างง่ายๆ)
pub fn validate_no_profanity(text: &str) -> Result<(), ValidationError> {
    let bad_words = ["spam", "abuse"]; // ในชีวิตจริงใช้ library ที่ดีกว่า
    
    let lower = text.to_lowercase();
    for word in &bad_words {
        if lower.contains(word) {
            let mut err = ValidationError::new("contains_profanity");
            err.message = Some(std::borrow::Cow::Borrowed("Content contains inappropriate language"));
            return Err(err);
        }
    }
    
    Ok(())
}

/// Validate JSON ว่าเป็น valid JSON object
pub fn validate_json_object(value: &str) -> Result<(), ValidationError> {
    match serde_json::from_str::<serde_json::Value>(value) {
        Ok(v) if v.is_object() => Ok(()),
        _ => {
            let mut err = ValidationError::new("invalid_json_object");
            err.message = Some(std::borrow::Cow::Borrowed("Must be a valid JSON object"));
            Err(err)
        }
    }
}
```

---

## 4. Nested Struct Validation

```rust
// src/validators/nested.rs
use serde::{Deserialize, Serialize};
use validator::{Validate, ValidationError};

#[derive(Debug, Deserialize, Validate, Serialize, Clone)]
pub struct Address {
    #[validate(length(min = 5, max = 200, message = "Street address required"))]
    pub street: String,
    
    #[validate(length(min = 2, max = 100, message = "City name required"))]
    pub city: String,
    
    #[validate(
        length(min = 2, max = 10),
        regex(path = "POSTAL_CODE_REGEX", message = "Invalid postal code")
    )]
    pub postal_code: String,
    
    #[validate(length(min = 2, max = 2, message = "Use 2-letter country code"))]
    pub country_code: String,
}

lazy_static::lazy_static! {
    static ref POSTAL_CODE_REGEX: regex::Regex = regex::Regex::new(r"^\d{5}$").unwrap();
}

#[derive(Debug, Deserialize, Validate, Serialize)]
pub struct CreateCompanyRequest {
    #[validate(length(min = 2, max = 255))]
    pub name: String,
    
    #[validate(email)]
    pub contact_email: String,
    
    #[validate]  // <- ทำให้ validate nested struct ด้วย
    pub billing_address: Address,
    
    #[validate]
    pub shipping_address: Option<Address>,
    
    #[validate(length(min = 1, max = 10))]
    pub departments: Vec<DepartmentInput>,
}

#[derive(Debug, Deserialize, Validate, Serialize, Clone)]
pub struct DepartmentInput {
    #[validate(length(min = 2, max = 100))]
    pub name: String,
    
    #[validate(range(min = 0, max = 10000))]
    pub budget: Option<f64>,
    
    #[validate(email)]
    pub manager_email: Option<String>,
}
```

---

## 5. Cross-field Validation

```rust
// src/validators/cross_field.rs
use serde::Deserialize;
use validator::{Validate, ValidationError, ValidationErrors};
use chrono::{NaiveDate, Utc};

#[derive(Debug, Deserialize)]
pub struct DateRangeRequest {
    pub start_date: NaiveDate,
    pub end_date: NaiveDate,
    pub max_duration_days: Option<i64>,
}

impl DateRangeRequest {
    /// Cross-field validation: end_date ต้องหลัง start_date
    pub fn validate_date_range(&self) -> Result<(), ValidationErrors> {
        let mut errors = ValidationErrors::new();
        
        if self.end_date <= self.start_date {
            let mut err = ValidationError::new("invalid_date_range");
            err.message = Some(std::borrow::Cow::Borrowed("end_date must be after start_date"));
            errors.add("end_date", err);
        }
        
        if let Some(max_days) = self.max_duration_days {
            let duration = (self.end_date - self.start_date).num_days();
            if duration > max_days {
                let mut err = ValidationError::new("date_range_too_long");
                err.message = Some(std::borrow::Cow::Owned(
                    format!("Date range cannot exceed {} days", max_days)
                ));
                errors.add("end_date", err);
            }
        }
        
        // ตรวจสอบว่าไม่ใช่ past dates สำหรับ booking
        if self.start_date < Utc::now().date_naive() {
            let mut err = ValidationError::new("past_date");
            err.message = Some(std::borrow::Cow::Borrowed("start_date cannot be in the past"));
            errors.add("start_date", err);
        }
        
        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
}

#[derive(Debug, Deserialize, Validate)]
pub struct PriceRangeRequest {
    #[validate(range(min = 0.0))]
    pub min_price: f64,
    
    #[validate(range(min = 0.0))]
    pub max_price: f64,
}

impl PriceRangeRequest {
    pub fn validate_price_range(&self) -> Result<(), ValidationErrors> {
        let mut errors = ValidationErrors::new();
        
        if self.max_price < self.min_price {
            let mut err = ValidationError::new("invalid_price_range");
            err.message = Some(std::borrow::Cow::Borrowed("max_price must be >= min_price"));
            errors.add("max_price", err);
        }
        
        if errors.is_empty() { Ok(()) } else { Err(errors) }
    }
}
```

---

## 6. Async Database Uniqueness Check

```rust
// src/validators/async_validation.rs
use sqlx::PgPool;
use validator::ValidationErrors;

pub struct AsyncValidator<'a> {
    pool: &'a PgPool,
    errors: ValidationErrors,
}

impl<'a> AsyncValidator<'a> {
    pub fn new(pool: &'a PgPool) -> Self {
        Self {
            pool,
            errors: ValidationErrors::new(),
        }
    }
    
    /// ตรวจสอบว่า email ไม่ซ้ำ
    pub async fn check_email_unique(&mut self, email: &str) -> &mut Self {
        match sqlx::query!(
            "SELECT id FROM users WHERE email = $1",
            email
        )
        .fetch_optional(self.pool)
        .await
        {
            Ok(Some(_)) => {
                let mut err = validator::ValidationError::new("email_taken");
                err.message = Some(std::borrow::Cow::Borrowed("Email is already registered"));
                self.errors.add("email", err);
            },
            Ok(None) => {},
            Err(e) => {
                log::error!("Database error checking email: {}", e);
                // ในกรณี error ให้ pass validation ไปก่อน
            }
        }
        self
    }
    
    /// ตรวจสอบว่า username ไม่ซ้ำ
    pub async fn check_username_unique(&mut self, username: &str) -> &mut Self {
        match sqlx::query!(
            "SELECT id FROM users WHERE username = $1",
            username
        )
        .fetch_optional(self.pool)
        .await
        {
            Ok(Some(_)) => {
                let mut err = validator::ValidationError::new("username_taken");
                err.message = Some(std::borrow::Cow::Borrowed("Username is already taken"));
                self.errors.add("username", err);
            },
            Ok(None) => {},
            Err(e) => log::error!("Database error: {}", e),
        }
        self
    }
    
    /// ตรวจสอบว่า slug ไม่ซ้ำสำหรับ blog posts
    pub async fn check_slug_unique(&mut self, slug: &str, exclude_id: Option<uuid::Uuid>) -> &mut Self {
        let result = if let Some(id) = exclude_id {
            sqlx::query!(
                "SELECT id FROM posts WHERE slug = $1 AND id != $2",
                slug, id
            )
            .fetch_optional(self.pool)
            .await
        } else {
            sqlx::query!(
                "SELECT id FROM posts WHERE slug = $1",
                slug
            )
            .fetch_optional(self.pool)
            .await
        };
        
        match result {
            Ok(Some(_)) => {
                let mut err = validator::ValidationError::new("slug_taken");
                err.message = Some(std::borrow::Cow::Borrowed("Slug is already in use"));
                self.errors.add("slug", err);
            },
            Ok(None) => {},
            Err(e) => log::error!("Database error: {}", e),
        }
        self
    }
    
    /// Return errors
    pub fn finish(self) -> Result<(), ValidationErrors> {
        if self.errors.is_empty() {
            Ok(())
        } else {
            Err(self.errors)
        }
    }
}
```

---

## 7. Input Sanitization

```rust
// src/sanitize.rs

/// Escape HTML entities เพื่อป้องกัน XSS
pub fn sanitize_html(input: &str) -> String {
    html_escape::encode_text(input).to_string()
}

/// Sanitize filename สำหรับ file uploads
pub fn sanitize_filename(filename: &str) -> String {
    sanitize_filename::sanitize(filename)
}

/// Trim whitespace และ normalize Unicode
pub fn sanitize_text(input: &str) -> String {
    input.trim().to_string()
}

/// ลบ SQL injection attempts (parameterized queries ดีกว่า แต่เพื่อ defense-in-depth)
pub fn check_sql_injection(input: &str) -> bool {
    let suspicious_patterns = [
        "--", "/*", "*/", "xp_", "exec(", "execute(",
        "select ", "insert ", "update ", "delete ", "drop ",
        "union ", "or 1=1", "or '1'='1",
    ];
    
    let lower = input.to_lowercase();
    suspicious_patterns.iter().any(|p| lower.contains(p))
}

/// Sanitize user input ทั้งหมด
#[derive(Debug)]
pub struct SanitizedInput {
    pub original: String,
    pub sanitized: String,
    pub was_modified: bool,
}

impl SanitizedInput {
    pub fn new(input: &str) -> Self {
        let trimmed = input.trim().to_string();
        let sanitized = sanitize_html(&trimmed);
        let was_modified = sanitized != trimmed;
        
        Self {
            original: input.to_string(),
            sanitized,
            was_modified,
        }
    }
}

/// Content sanitization สำหรับ Rich Text
pub fn sanitize_rich_text(html: &str) -> String {
    // ในชีวิตจริงควรใช้ ammonia หรือ library ที่ดี
    // นี่เป็น example แบบง่าย
    let mut result = String::new();
    let allowed_tags = ["<b>", "</b>", "<i>", "</i>", "<p>", "</p>", "<br>"];
    
    // Simple approach: escape everything แล้ว restore allowed tags
    let escaped = html_escape::encode_text(html).to_string();
    
    // ในชีวิตจริงใช้ proper HTML parser
    escaped
}
```

---

## 8. File Upload Validation

```rust
// src/validators/file_upload.rs
use actix_multipart::Multipart;
use actix_web::Error;
use futures::{StreamExt, TryStreamExt};
use std::io::Write;
use mime::Mime;

#[derive(Debug)]
pub struct UploadedFile {
    pub filename: String,
    pub content_type: String,
    pub size_bytes: usize,
    pub data: Vec<u8>,
}

#[derive(Debug, Clone)]
pub struct FileUploadConfig {
    pub max_size_bytes: usize,
    pub allowed_mime_types: Vec<String>,
    pub allowed_extensions: Vec<String>,
}

impl FileUploadConfig {
    pub fn images() -> Self {
        Self {
            max_size_bytes: 5 * 1024 * 1024, // 5MB
            allowed_mime_types: vec![
                "image/jpeg".to_string(),
                "image/png".to_string(),
                "image/gif".to_string(),
                "image/webp".to_string(),
            ],
            allowed_extensions: vec![
                "jpg".to_string(), "jpeg".to_string(),
                "png".to_string(), "gif".to_string(),
                "webp".to_string(),
            ],
        }
    }
    
    pub fn documents() -> Self {
        Self {
            max_size_bytes: 10 * 1024 * 1024, // 10MB
            allowed_mime_types: vec![
                "application/pdf".to_string(),
                "application/msword".to_string(),
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document".to_string(),
                "text/plain".to_string(),
            ],
            allowed_extensions: vec![
                "pdf".to_string(), "doc".to_string(),
                "docx".to_string(), "txt".to_string(),
            ],
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum FileUploadError {
    #[error("File too large: {size} bytes, max: {max} bytes")]
    TooLarge { size: usize, max: usize },
    
    #[error("Invalid file type: {mime_type}")]
    InvalidMimeType { mime_type: String },
    
    #[error("Invalid file extension: {extension}")]
    InvalidExtension { extension: String },
    
    #[error("Empty filename")]
    EmptyFilename,
    
    #[error("IO error: {0}")]
    IoError(#[from] std::io::Error),
}

pub fn validate_file(
    file: &UploadedFile,
    config: &FileUploadConfig,
) -> Result<(), FileUploadError> {
    // ตรวจสอบ size
    if file.size_bytes > config.max_size_bytes {
        return Err(FileUploadError::TooLarge {
            size: file.size_bytes,
            max: config.max_size_bytes,
        });
    }
    
    // ตรวจสอบ filename
    if file.filename.is_empty() {
        return Err(FileUploadError::EmptyFilename);
    }
    
    // ตรวจสอบ extension
    let extension = std::path::Path::new(&file.filename)
        .extension()
        .and_then(|e| e.to_str())
        .map(|e| e.to_lowercase())
        .unwrap_or_default();
    
    if !config.allowed_extensions.contains(&extension) {
        return Err(FileUploadError::InvalidExtension { extension });
    }
    
    // ตรวจสอบ MIME type
    if !config.allowed_mime_types.contains(&file.content_type) {
        return Err(FileUploadError::InvalidMimeType {
            mime_type: file.content_type.clone(),
        });
    }
    
    // ตรวจสอบ magic bytes (file signature)
    validate_file_magic_bytes(&file.data, &file.content_type)?;
    
    Ok(())
}

/// ตรวจสอบ magic bytes ของไฟล์
fn validate_file_magic_bytes(
    data: &[u8],
    claimed_mime: &str,
) -> Result<(), FileUploadError> {
    if data.len() < 4 {
        return Ok(()); // ไฟล์เล็กเกินไป ข้ามไป
    }
    
    let valid = match claimed_mime {
        "image/jpeg" => data.starts_with(&[0xFF, 0xD8, 0xFF]),
        "image/png" => data.starts_with(&[0x89, 0x50, 0x4E, 0x47]),
        "image/gif" => data.starts_with(b"GIF8"),
        "application/pdf" => data.starts_with(b"%PDF"),
        _ => true, // ไม่รู้จัก type → ให้ผ่าน
    };
    
    if !valid {
        return Err(FileUploadError::InvalidMimeType {
            mime_type: format!("{} (magic bytes mismatch)", claimed_mime),
        });
    }
    
    Ok(())
}

pub async fn process_multipart(
    mut payload: Multipart,
    config: &FileUploadConfig,
) -> Result<Vec<UploadedFile>, Error> {
    let mut files = Vec::new();
    
    while let Ok(Some(mut field)) = payload.try_next().await {
        let content_disposition = field.content_disposition();
        
        let filename = content_disposition
            .get_filename()
            .map(|f| sanitize_filename::sanitize(f))
            .unwrap_or_else(|| uuid::Uuid::new_v4().to_string());
        
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_else(|| "application/octet-stream".to_string());
        
        let mut data = Vec::new();
        while let Some(chunk) = field.next().await {
            let chunk = chunk.map_err(|e| actix_web::error::ErrorBadRequest(e))?;
            
            // ตรวจสอบ size ระหว่าง streaming
            if data.len() + chunk.len() > config.max_size_bytes {
                return Err(actix_web::error::ErrorPayloadTooLarge(
                    format!("File too large, max {} bytes", config.max_size_bytes)
                ));
            }
            
            data.extend_from_slice(&chunk);
        }
        
        let file = UploadedFile {
            filename: filename.clone(),
            content_type: content_type.clone(),
            size_bytes: data.len(),
            data,
        };
        
        validate_file(&file, config)
            .map_err(|e| actix_web::error::ErrorBadRequest(e.to_string()))?;
        
        files.push(file);
    }
    
    Ok(files)
}
```

---

## 9. Complete Registration Validation

```rust
// src/handlers/register.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use validator::Validate;

use crate::{
    validators::{
        custom::*,
        async_validation::AsyncValidator,
    },
    sanitize::sanitize_text,
};

#[derive(Debug, Deserialize, Validate, Serialize)]
pub struct FullRegisterRequest {
    #[validate(
        length(min = 3, max = 30, message = "Username must be 3-30 characters"),
        custom = "validate_username_not_reserved",
        regex(path = "USERNAME_REGEX", message = "Username: letters, numbers, underscores only")
    )]
    pub username: String,
    
    #[validate(
        email(message = "Invalid email format"),
        length(max = 255)
    )]
    pub email: String,
    
    #[validate(
        length(min = 8, max = 128, message = "Password must be 8-128 characters")
    )]
    pub password: String,
    
    #[validate(must_match(other = "password", message = "Passwords do not match"))]
    pub confirm_password: String,
    
    #[validate(
        length(min = 1, max = 100, message = "First name required"),
        custom = "validate_no_profanity"
    )]
    pub first_name: String,
    
    #[validate(
        length(min = 1, max = 100, message = "Last name required"),
        custom = "validate_no_profanity"
    )]
    pub last_name: String,
    
    #[validate(range(min = 13, max = 120, message = "Age must be between 13 and 120"))]
    pub age: Option<u8>,
    
    #[validate(url(message = "Invalid website URL"))]
    pub website: Option<String>,
    
    pub accept_terms: bool,
}

lazy_static::lazy_static! {
    static ref USERNAME_REGEX: regex::Regex = regex::Regex::new(r"^[a-zA-Z0-9_]+$").unwrap();
}

#[derive(Debug, Serialize)]
pub struct ValidationErrorResponse {
    pub errors: std::collections::HashMap<String, Vec<String>>,
}

pub async fn register(
    pool: web::Data<PgPool>,
    req: web::Json<FullRegisterRequest>,
) -> actix_web::Result<HttpResponse> {
    // 1. ตรวจสอบ terms acceptance
    if !req.accept_terms {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "You must accept the terms of service"
        })));
    }
    
    // 2. Sanitize input ก่อน validate
    let sanitized_username = sanitize_text(&req.username);
    let sanitized_email = sanitize_text(&req.email).to_lowercase();
    let sanitized_first_name = sanitize_text(&req.first_name);
    let sanitized_last_name = sanitize_text(&req.last_name);
    
    // 3. Struct-level validation
    if let Err(errors) = req.validate() {
        let mut error_map: std::collections::HashMap<String, Vec<String>> = std::collections::HashMap::new();
        
        for (field, field_errors) in errors.field_errors() {
            let messages: Vec<String> = field_errors.iter()
                .map(|e| e.message.as_ref()
                    .map(|m| m.to_string())
                    .unwrap_or_else(|| format!("Validation failed: {}", e.code)))
                .collect();
            error_map.insert(field.to_string(), messages);
        }
        
        return Ok(HttpResponse::UnprocessableEntity().json(ValidationErrorResponse {
            errors: error_map,
        }));
    }
    
    // 4. Async database uniqueness checks
    let validation_result = AsyncValidator::new(pool.get_ref())
        .check_email_unique(&sanitized_email)
        .await
        .check_username_unique(&sanitized_username)
        .await
        .finish();
    
    if let Err(errors) = validation_result {
        let mut error_map: std::collections::HashMap<String, Vec<String>> = std::collections::HashMap::new();
        
        for (field, field_errors) in errors.field_errors() {
            let messages: Vec<String> = field_errors.iter()
                .map(|e| e.message.as_ref()
                    .map(|m| m.to_string())
                    .unwrap_or_default())
                .collect();
            error_map.insert(field.to_string(), messages);
        }
        
        return Ok(HttpResponse::Conflict().json(ValidationErrorResponse {
            errors: error_map,
        }));
    }
    
    // 5. Hash password
    let password_hash = bcrypt::hash(&req.password, bcrypt::DEFAULT_COST)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    // 6. สร้าง user
    let user_id = sqlx::query!(
        r#"INSERT INTO users (username, email, password_hash, first_name, last_name)
           VALUES ($1, $2, $3, $4, $5)
           RETURNING id"#,
        sanitized_username,
        sanitized_email,
        password_hash,
        sanitized_first_name,
        sanitized_last_name,
    )
    .fetch_one(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?
    .id;
    
    Ok(HttpResponse::Created().json(serde_json::json!({
        "message": "Registration successful",
        "user_id": user_id,
        "email": sanitized_email,
    })))
}
```

---

## 10. Error Response Helper

```rust
// src/error.rs
use actix_web::HttpResponse;
use validator::ValidationErrors;
use std::collections::HashMap;

pub fn validation_error_response(errors: ValidationErrors) -> HttpResponse {
    let mut error_map: HashMap<String, Vec<String>> = HashMap::new();
    
    for (field, field_errors) in errors.field_errors() {
        let messages: Vec<String> = field_errors.iter()
            .map(|e| {
                e.message.as_ref()
                    .map(|m| m.to_string())
                    .unwrap_or_else(|| {
                        match e.code.as_ref() {
                            "length" => "Invalid length".to_string(),
                            "email" => "Invalid email format".to_string(),
                            "range" => "Value out of range".to_string(),
                            "must_match" => "Fields do not match".to_string(),
                            "url" => "Invalid URL format".to_string(),
                            code => format!("Validation error: {}", code),
                        }
                    })
            })
            .collect();
        error_map.insert(field.to_string(), messages);
    }
    
    HttpResponse::UnprocessableEntity().json(serde_json::json!({
        "error": "Validation failed",
        "details": error_map
    }))
}
```

---

## 11. สรุปสิ่งที่เรียนรู้

✅ validator crate พร้อม built-in validators  
✅ Custom validator functions  
✅ Nested struct validation ด้วย #[validate]  
✅ Cross-field validation  
✅ Async database uniqueness checks  
✅ HTML sanitization ป้องกัน XSS  
✅ File upload validation พร้อม magic bytes  
✅ Complete registration flow  
✅ Proper error response format  

---

*[← Part 047: TLS and HTTPS](../part_047/README.md) | [Part 049: Security Audit →](../part_049/README.md)*
