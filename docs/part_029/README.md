# Part 029: Form Data and Multipart ใน Actix-web

## สารบัญ
- [แนะนำ Form Data](#แนะนำ-form-data)
- [HTML Form Handling (application/x-www-form-urlencoded)](#html-form-handling)
- [Multipart Form Data](#multipart-form-data)
- [actix-multipart Crate](#actix-multipart-crate)
- [File Upload Handling](#file-upload-handling)
- [Saving Uploaded Files](#saving-uploaded-files)
- [File Size Validation](#file-size-validation)
- [MIME Type Validation](#mime-type-validation)
- [Multiple File Upload](#multiple-file-upload)

---

## แนะนำ Form Data

Web forms ส่งข้อมูลได้สองรูปแบบหลัก:
1. `application/x-www-form-urlencoded` - สำหรับข้อมูล text ทั่วไป
2. `multipart/form-data` - สำหรับ file uploads

### Dependencies

```toml
[dependencies]
actix-web = "4"
actix-multipart = "0.6"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
futures-util = "0.3"
uuid = { version = "1", features = ["v4"] }
mime = "0.3"
mime_guess = "2"
sanitize-filename = "0.5"
bytes = "1"
```

---

## HTML Form Handling

`application/x-www-form-urlencoded` คือ format ที่ browser ใช้โดย default เมื่อ submit form

```rust
use actix_web::{web, HttpResponse, Result};
use serde::{Deserialize, Serialize};

// Form structs
#[derive(Debug, Deserialize)]
struct LoginForm {
    username: String,
    password: String,
    remember_me: Option<String>,  // checkbox ส่งมาเป็น "on" หรือ None
}

#[derive(Debug, Deserialize)]
struct RegisterForm {
    first_name: String,
    last_name: String,
    email: String,
    password: String,
    password_confirm: String,
    date_of_birth: String,
    gender: Option<String>,
    phone: Option<String>,
    agree_terms: Option<String>,  // checkbox
    newsletter: Option<String>,   // checkbox
}

#[derive(Debug, Deserialize)]
struct ProfileUpdateForm {
    bio: Option<String>,
    website: Option<String>,
    location: Option<String>,
    twitter: Option<String>,
    github: Option<String>,
    linkedin: Option<String>,
}

#[derive(Debug, Deserialize)]
struct SearchForm {
    query: String,
    category: Option<String>,
    sort: Option<String>,
    order: Option<String>,
    page: Option<u32>,
}

// Handlers
async fn login_handler(form: web::Form<LoginForm>) -> Result<HttpResponse> {
    let data = form.into_inner();
    
    // Validate
    if data.username.trim().is_empty() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Username is required"
        })));
    }
    
    if data.password.len() < 6 {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Password must be at least 6 characters"
        })));
    }
    
    let remember = data.remember_me.as_deref() == Some("on");
    
    // Authenticate (simplified)
    if data.username == "admin" && data.password == "password123" {
        Ok(HttpResponse::Ok().json(serde_json::json!({
            "success": true,
            "message": "Login successful",
            "remember_me": remember,
            "token": "mock-jwt-token"
        })))
    } else {
        Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid credentials"
        })))
    }
}

async fn register_handler(form: web::Form<RegisterForm>) -> Result<HttpResponse> {
    let data = form.into_inner();
    
    // Validation
    let mut errors: Vec<String> = vec![];
    
    if data.first_name.trim().is_empty() {
        errors.push("First name is required".to_string());
    }
    if data.last_name.trim().is_empty() {
        errors.push("Last name is required".to_string());
    }
    if !data.email.contains('@') {
        errors.push("Invalid email format".to_string());
    }
    if data.password.len() < 8 {
        errors.push("Password must be at least 8 characters".to_string());
    }
    if data.password != data.password_confirm {
        errors.push("Passwords do not match".to_string());
    }
    if data.agree_terms.is_none() {
        errors.push("You must agree to the terms".to_string());
    }
    
    if !errors.is_empty() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "success": false,
            "errors": errors
        })));
    }
    
    let subscribed_newsletter = data.newsletter.as_deref() == Some("on");
    
    Ok(HttpResponse::Created().json(serde_json::json!({
        "success": true,
        "message": "Registration successful",
        "user": {
            "email": data.email,
            "name": format!("{} {}", data.first_name, data.last_name),
            "newsletter": subscribed_newsletter
        }
    })))
}

async fn search_handler(form: web::Form<SearchForm>) -> Result<HttpResponse> {
    let data = form.into_inner();
    
    if data.query.trim().is_empty() {
        return Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Search query is required"
        })));
    }
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "query": data.query,
        "category": data.category,
        "sort": data.sort.unwrap_or_else(|| "relevance".to_string()),
        "order": data.order.unwrap_or_else(|| "desc".to_string()),
        "page": data.page.unwrap_or(1),
        "results": []  // ผลลัพธ์จาก database
    })))
}

// Form จาก Query String ก็ใช้ได้เช่นกัน
async fn search_get_handler(
    query: web::Query<SearchForm>,
) -> Result<HttpResponse> {
    let data = query.into_inner();
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "query": data.query,
        "category": data.category
    })))
}
```

---

## Multipart Form Data

`multipart/form-data` ใช้สำหรับ upload files หรือส่งข้อมูลผสมระหว่าง text และ binary

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Result, Error};
use futures_util::TryStreamExt;
use tokio::io::AsyncWriteExt;
use std::collections::HashMap;

// โครงสร้างสำหรับเก็บข้อมูลที่ parse จาก multipart
#[derive(Debug)]
struct MultipartData {
    fields: HashMap<String, String>,
    files: Vec<UploadedFile>,
}

#[derive(Debug)]
struct UploadedFile {
    field_name: String,
    filename: String,
    content_type: String,
    size: usize,
    data: Vec<u8>,
}

// Parse multipart form อย่างง่าย
async fn parse_multipart(mut payload: Multipart) -> Result<MultipartData, Error> {
    let mut fields = HashMap::new();
    let mut files = vec![];
    
    while let Some(mut field) = payload.try_next().await? {
        let content_disposition = field.content_disposition();
        let field_name = content_disposition
            .get_name()
            .unwrap_or("unknown")
            .to_string();
        
        // ตรวจสอบว่าเป็น file field หรือ text field
        if let Some(filename) = content_disposition.get_filename() {
            // File field
            let filename = filename.to_string();
            let content_type = field.content_type()
                .map(|ct| ct.to_string())
                .unwrap_or_else(|| "application/octet-stream".to_string());
            
            let mut data = vec![];
            while let Some(chunk) = field.try_next().await? {
                data.extend_from_slice(&chunk);
            }
            
            let size = data.len();
            files.push(UploadedFile {
                field_name,
                filename,
                content_type,
                size,
                data,
            });
        } else {
            // Text field
            let mut value_bytes = vec![];
            while let Some(chunk) = field.try_next().await? {
                value_bytes.extend_from_slice(&chunk);
            }
            
            if let Ok(value) = String::from_utf8(value_bytes) {
                fields.insert(field_name, value);
            }
        }
    }
    
    Ok(MultipartData { fields, files })
}
```

---

## actix-multipart Crate

การใช้ actix-multipart อย่างละเอียด

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Result, Error};
use futures_util::{StreamExt, TryStreamExt};
use bytes::Bytes;

// Upload single file พร้อม metadata
async fn upload_with_metadata(mut payload: Multipart) -> Result<HttpResponse, Error> {
    let mut title = String::new();
    let mut description = String::new();
    let mut file_data: Option<(String, String, Vec<u8>)> = None;
    
    while let Some(mut field) = payload.try_next().await? {
        let name = field.content_disposition()
            .get_name()
            .unwrap_or("")
            .to_string();
        
        match name.as_str() {
            "title" => {
                let mut bytes = vec![];
                while let Some(chunk) = field.try_next().await? {
                    bytes.extend_from_slice(&chunk);
                }
                title = String::from_utf8_lossy(&bytes).to_string();
            },
            "description" => {
                let mut bytes = vec![];
                while let Some(chunk) = field.try_next().await? {
                    bytes.extend_from_slice(&chunk);
                }
                description = String::from_utf8_lossy(&bytes).to_string();
            },
            "file" => {
                let filename = field.content_disposition()
                    .get_filename()
                    .unwrap_or("unknown")
                    .to_string();
                let content_type = field.content_type()
                    .map(|ct| ct.to_string())
                    .unwrap_or_else(|| "application/octet-stream".to_string());
                
                let mut data = vec![];
                while let Some(chunk) = field.try_next().await? {
                    data.extend_from_slice(&chunk);
                }
                
                file_data = Some((filename, content_type, data));
            },
            _ => {
                // Skip unknown fields
                while field.try_next().await?.is_some() {}
            }
        }
    }
    
    match file_data {
        Some((filename, content_type, data)) => {
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "success": true,
                "title": title,
                "description": description,
                "file": {
                    "name": filename,
                    "content_type": content_type,
                    "size": data.len()
                }
            })))
        },
        None => {
            Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "No file provided"
            })))
        }
    }
}
```

---

## File Upload Handling

การจัดการ file upload อย่างสมบูรณ์

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Result, Error};
use futures_util::TryStreamExt;
use std::path::{Path, PathBuf};
use uuid::Uuid;

// Upload configuration
#[derive(Clone)]
struct UploadConfig {
    upload_dir: PathBuf,
    max_file_size: usize,           // bytes
    allowed_types: Vec<String>,
    max_files_per_request: usize,
}

impl UploadConfig {
    fn for_images() -> Self {
        UploadConfig {
            upload_dir: PathBuf::from("storage/uploads/images"),
            max_file_size: 5 * 1024 * 1024,  // 5MB
            allowed_types: vec![
                "image/jpeg".to_string(),
                "image/png".to_string(),
                "image/gif".to_string(),
                "image/webp".to_string(),
            ],
            max_files_per_request: 10,
        }
    }
    
    fn for_documents() -> Self {
        UploadConfig {
            upload_dir: PathBuf::from("storage/uploads/documents"),
            max_file_size: 20 * 1024 * 1024,  // 20MB
            allowed_types: vec![
                "application/pdf".to_string(),
                "application/msword".to_string(),
                "application/vnd.openxmlformats-officedocument.wordprocessingml.document".to_string(),
                "application/vnd.ms-excel".to_string(),
                "text/plain".to_string(),
                "text/csv".to_string(),
            ],
            max_files_per_request: 5,
        }
    }
}

// Upload result
#[derive(Debug, serde::Serialize)]
struct FileUploadResult {
    original_name: String,
    stored_name: String,
    path: String,
    url: String,
    size: usize,
    content_type: String,
}

#[derive(Debug, serde::Serialize)]
struct UploadResponse {
    success: bool,
    files: Vec<FileUploadResult>,
    failed: Vec<UploadError>,
    total: usize,
    uploaded: usize,
}

#[derive(Debug, serde::Serialize)]
struct UploadError {
    filename: String,
    error: String,
}

// Sanitize filename เพื่อความปลอดภัย
fn sanitize_filename(filename: &str) -> String {
    // ลบ path separators และ characters ที่อันตราย
    let sanitized: String = filename
        .chars()
        .filter(|c| c.is_alphanumeric() || *c == '.' || *c == '-' || *c == '_')
        .collect();
    
    // ป้องกัน empty filename
    if sanitized.is_empty() {
        return "unnamed_file".to_string();
    }
    
    // ป้องกัน hidden files (เริ่มด้วย .)
    if sanitized.starts_with('.') {
        return format!("file{}", sanitized);
    }
    
    sanitized
}

// สร้าง unique filename
fn generate_unique_filename(original_name: &str) -> String {
    let ext = Path::new(original_name)
        .extension()
        .and_then(|e| e.to_str())
        .unwrap_or("");
    
    if ext.is_empty() {
        Uuid::new_v4().to_string()
    } else {
        format!("{}.{}", Uuid::new_v4(), ext.to_lowercase())
    }
}

// Validate file ก่อน save
fn validate_file(
    filename: &str,
    content_type: &str,
    data: &[u8],
    config: &UploadConfig,
) -> Result<(), String> {
    // ตรวจสอบ size
    if data.len() > config.max_file_size {
        return Err(format!(
            "File size {} bytes exceeds maximum {} bytes",
            data.len(),
            config.max_file_size
        ));
    }
    
    // ตรวจสอบ content type
    if !config.allowed_types.contains(&content_type.to_string()) {
        return Err(format!(
            "File type '{}' is not allowed",
            content_type
        ));
    }
    
    // ตรวจสอบ magic bytes (file signature)
    validate_file_magic_bytes(content_type, data)?;
    
    // ตรวจสอบ extension
    let ext = Path::new(filename)
        .extension()
        .and_then(|e| e.to_str())
        .unwrap_or("")
        .to_lowercase();
    
    let allowed_extensions = get_allowed_extensions(content_type);
    if !allowed_extensions.is_empty() && !allowed_extensions.contains(&ext.as_str()) {
        return Err(format!(
            "File extension '.{}' does not match content type '{}'",
            ext, content_type
        ));
    }
    
    Ok(())
}

// ตรวจสอบ magic bytes ของไฟล์
fn validate_file_magic_bytes(content_type: &str, data: &[u8]) -> Result<(), String> {
    if data.len() < 4 {
        return Err("File is too small to be valid".to_string());
    }
    
    let valid = match content_type {
        "image/jpeg" => data.starts_with(&[0xFF, 0xD8, 0xFF]),
        "image/png" => data.starts_with(&[0x89, 0x50, 0x4E, 0x47]),
        "image/gif" => data.starts_with(b"GIF87a") || data.starts_with(b"GIF89a"),
        "image/webp" => data.len() >= 12 && &data[0..4] == b"RIFF" && &data[8..12] == b"WEBP",
        "application/pdf" => data.starts_with(b"%PDF"),
        _ => true,  // ไม่ตรวจสอบสำหรับ types อื่น
    };
    
    if valid {
        Ok(())
    } else {
        Err(format!("File content does not match declared type '{}'", content_type))
    }
}

// แปลง content type เป็น allowed extensions
fn get_allowed_extensions(content_type: &str) -> Vec<&'static str> {
    match content_type {
        "image/jpeg" => vec!["jpg", "jpeg"],
        "image/png" => vec!["png"],
        "image/gif" => vec!["gif"],
        "image/webp" => vec!["webp"],
        "application/pdf" => vec!["pdf"],
        "text/plain" => vec!["txt"],
        "text/csv" => vec!["csv"],
        _ => vec![],
    }
}
```

---

## Saving Uploaded Files

การบันทึกไฟล์ที่ upload ลงบน disk

```rust
use tokio::fs;
use tokio::io::AsyncWriteExt;
use std::path::PathBuf;

// บันทึกไฟล์ลง disk
async fn save_uploaded_file(
    data: &[u8],
    dir: &PathBuf,
    filename: &str,
) -> Result<PathBuf, std::io::Error> {
    // สร้าง directory ถ้าไม่มี
    fs::create_dir_all(dir).await?;
    
    let file_path = dir.join(filename);
    
    // เขียนไฟล์
    let mut file = tokio::fs::File::create(&file_path).await?;
    file.write_all(data).await?;
    file.flush().await?;
    
    Ok(file_path)
}

// Handler ที่ upload และ save ไฟล์
async fn upload_image(
    config: web::Data<UploadConfig>,
    mut payload: Multipart,
) -> Result<HttpResponse, Error> {
    let mut results: Vec<FileUploadResult> = vec![];
    let mut errors: Vec<UploadError> = vec![];
    
    while let Some(mut field) = payload.try_next().await? {
        let name = field.content_disposition()
            .get_name()
            .unwrap_or("")
            .to_string();
        
        if name != "file" && name != "files" {
            // Skip non-file fields
            while field.try_next().await?.is_some() {}
            continue;
        }
        
        let original_name = field.content_disposition()
            .get_filename()
            .map(|f| sanitize_filename(f))
            .unwrap_or_else(|| "unnamed".to_string());
        
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_else(|| "application/octet-stream".to_string());
        
        // อ่านข้อมูลทั้งหมด
        let mut data: Vec<u8> = vec![];
        let mut size_exceeded = false;
        
        while let Some(chunk) = field.try_next().await? {
            data.extend_from_slice(&chunk);
            
            // ตรวจสอบ size ขณะอ่าน
            if data.len() > config.max_file_size {
                size_exceeded = true;
                break;
            }
        }
        
        if size_exceeded {
            errors.push(UploadError {
                filename: original_name,
                error: format!("File exceeds maximum size of {} MB", 
                    config.max_file_size / 1024 / 1024),
            });
            continue;
        }
        
        // Validate file
        if let Err(err) = validate_file(&original_name, &content_type, &data, &config) {
            errors.push(UploadError {
                filename: original_name,
                error: err,
            });
            continue;
        }
        
        // สร้าง unique filename
        let stored_name = generate_unique_filename(&original_name);
        
        // สร้าง subdirectory ตาม content type
        let sub_dir = match content_type.as_str() {
            ct if ct.starts_with("image/") => "images",
            "application/pdf" => "pdfs",
            _ => "other",
        };
        
        let upload_path = config.upload_dir.join(sub_dir);
        
        // บันทึกไฟล์
        match save_uploaded_file(&data, &upload_path, &stored_name).await {
            Ok(file_path) => {
                let relative_path = file_path.to_string_lossy().to_string();
                let url = format!("/uploads/{}/{}", sub_dir, stored_name);
                
                results.push(FileUploadResult {
                    original_name: original_name.clone(),
                    stored_name,
                    path: relative_path,
                    url,
                    size: data.len(),
                    content_type,
                });
            },
            Err(e) => {
                log::error!("Failed to save file {}: {}", original_name, e);
                errors.push(UploadError {
                    filename: original_name,
                    error: "Failed to save file".to_string(),
                });
            }
        }
    }
    
    let total = results.len() + errors.len();
    let uploaded = results.len();
    
    let response = UploadResponse {
        success: errors.is_empty(),
        files: results,
        failed: errors,
        total,
        uploaded,
    };
    
    if uploaded == 0 {
        Ok(HttpResponse::BadRequest().json(response))
    } else if !response.failed.is_empty() {
        Ok(HttpResponse::MultiStatus().json(response))
    } else {
        Ok(HttpResponse::Ok().json(response))
    }
}
```

---

## File Size Validation

การตรวจสอบขนาดไฟล์

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Result, Error};
use futures_util::TryStreamExt;

// Constants
const MAX_IMAGE_SIZE: usize = 5 * 1024 * 1024;    // 5MB
const MAX_DOCUMENT_SIZE: usize = 20 * 1024 * 1024; // 20MB
const MAX_VIDEO_SIZE: usize = 100 * 1024 * 1024;   // 100MB

// Middleware สำหรับ limit payload size
use actix_web::web::PayloadConfig;

fn configure_upload_limits(cfg: &mut web::ServiceConfig) {
    // สำหรับ image upload
    let image_payload = PayloadConfig::new(MAX_IMAGE_SIZE);
    
    // สำหรับ document upload
    let doc_payload = PayloadConfig::new(MAX_DOCUMENT_SIZE);
    
    cfg
        .service(
            web::resource("/upload/image")
                .app_data(image_payload.clone())
                .route(web::post().to(upload_single_image))
        )
        .service(
            web::resource("/upload/document")
                .app_data(doc_payload)
                .route(web::post().to(upload_document))
        );
}

// อ่านไฟล์ทีละ chunk เพื่อตรวจสอบขนาด
async fn upload_with_size_check(
    mut payload: Multipart,
    max_size: usize,
) -> Result<Vec<u8>, String> {
    let mut data: Vec<u8> = Vec::new();
    
    if let Some(mut field) = payload.try_next().await.map_err(|e| e.to_string())? {
        while let Some(chunk) = field.try_next().await.map_err(|e| e.to_string())? {
            data.extend_from_slice(&chunk);
            
            if data.len() > max_size {
                return Err(format!(
                    "File size exceeds maximum {} MB",
                    max_size / 1024 / 1024
                ));
            }
        }
    }
    
    Ok(data)
}

async fn upload_single_image(mut payload: Multipart) -> Result<HttpResponse, Error> {
    let mut uploaded = None;
    
    while let Some(mut field) = payload.try_next().await? {
        if field.content_disposition().get_filename().is_none() {
            while field.try_next().await?.is_some() {}
            continue;
        }
        
        let filename = field.content_disposition()
            .get_filename()
            .map(sanitize_filename)
            .unwrap_or_else(|| "image.jpg".to_string());
        
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_default();
        
        // ตรวจสอบ content type ก่อน
        if !content_type.starts_with("image/") {
            return Ok(HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Only image files are allowed"
            })));
        }
        
        let mut data: Vec<u8> = vec![];
        
        while let Some(chunk) = field.try_next().await? {
            data.extend_from_slice(&chunk);
            
            if data.len() > MAX_IMAGE_SIZE {
                return Ok(HttpResponse::PayloadTooLarge().json(serde_json::json!({
                    "error": format!("Image must be smaller than {} MB", MAX_IMAGE_SIZE / 1024 / 1024)
                })));
            }
        }
        
        uploaded = Some((filename, content_type, data));
        break;  // รับแค่ไฟล์เดียว
    }
    
    match uploaded {
        Some((filename, content_type, data)) => {
            // Process image...
            Ok(HttpResponse::Ok().json(serde_json::json!({
                "success": true,
                "filename": filename,
                "content_type": content_type,
                "size": data.len(),
                "size_kb": data.len() / 1024
            })))
        },
        None => Ok(HttpResponse::BadRequest().json(serde_json::json!({
            "error": "No image file provided"
        }))),
    }
}

async fn upload_document(mut payload: Multipart) -> Result<HttpResponse, Error> {
    // Similar to upload_single_image but with different size limit and type checks
    Ok(HttpResponse::Ok().body("Document uploaded"))
}
```

---

## MIME Type Validation

การตรวจสอบ MIME type อย่างถูกต้อง

```rust
use std::collections::HashSet;

// Allowed types สำหรับแต่ละ category
fn get_allowed_image_types() -> HashSet<&'static str> {
    let mut types = HashSet::new();
    types.insert("image/jpeg");
    types.insert("image/jpg");
    types.insert("image/png");
    types.insert("image/gif");
    types.insert("image/webp");
    types.insert("image/svg+xml");
    types.insert("image/bmp");
    types.insert("image/tiff");
    types
}

fn get_allowed_document_types() -> HashSet<&'static str> {
    let mut types = HashSet::new();
    types.insert("application/pdf");
    types.insert("application/msword");
    types.insert("application/vnd.openxmlformats-officedocument.wordprocessingml.document");
    types.insert("application/vnd.ms-excel");
    types.insert("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    types.insert("application/vnd.ms-powerpoint");
    types.insert("text/plain");
    types.insert("text/csv");
    types
}

// Validate ด้วย magic bytes และ MIME type
struct FileValidator {
    allowed_types: HashSet<String>,
    max_size: usize,
}

impl FileValidator {
    fn images(max_size: usize) -> Self {
        FileValidator {
            allowed_types: get_allowed_image_types()
                .iter()
                .map(|s| s.to_string())
                .collect(),
            max_size,
        }
    }
    
    fn validate(&self, filename: &str, content_type: &str, data: &[u8]) -> Result<(), String> {
        // 1. ตรวจสอบขนาด
        if data.len() > self.max_size {
            return Err(format!(
                "File size ({} KB) exceeds maximum ({} KB)",
                data.len() / 1024,
                self.max_size / 1024
            ));
        }
        
        // 2. ตรวจสอบ MIME type จาก declared content type
        if !self.allowed_types.contains(content_type) {
            return Err(format!(
                "File type '{}' is not allowed. Allowed: {:?}",
                content_type,
                self.allowed_types
            ));
        }
        
        // 3. ตรวจสอบ magic bytes
        let detected_type = detect_file_type(data);
        if let Some(detected) = detected_type {
            if detected != content_type && !is_compatible_type(detected, content_type) {
                return Err(format!(
                    "File content ({}) does not match declared type ({})",
                    detected, content_type
                ));
            }
        }
        
        // 4. ตรวจสอบ extension กับ MIME type
        validate_extension_mime(filename, content_type)?;
        
        Ok(())
    }
}

// Detect file type จาก magic bytes
fn detect_file_type(data: &[u8]) -> Option<&'static str> {
    if data.len() < 4 {
        return None;
    }
    
    match data {
        d if d.starts_with(&[0xFF, 0xD8, 0xFF]) => Some("image/jpeg"),
        d if d.starts_with(&[0x89, 0x50, 0x4E, 0x47]) => Some("image/png"),
        d if d.starts_with(b"GIF87a") || d.starts_with(b"GIF89a") => Some("image/gif"),
        d if d.len() >= 12 && &d[0..4] == b"RIFF" && &d[8..12] == b"WEBP" => Some("image/webp"),
        d if d.starts_with(b"%PDF") => Some("application/pdf"),
        d if d.starts_with(&[0x50, 0x4B, 0x03, 0x04]) => Some("application/zip"),
        d if d.starts_with(&[0xD0, 0xCF, 0x11, 0xE0]) => Some("application/msword"),
        _ => None,
    }
}

// ตรวจสอบว่า types compatible กัน
fn is_compatible_type(detected: &str, declared: &str) -> bool {
    // บาง types สามารถใช้แทนกันได้
    matches!((detected, declared),
        ("image/jpeg", "image/jpg") |
        ("image/jpg", "image/jpeg") |
        ("application/zip", "application/vnd.openxmlformats-officedocument.wordprocessingml.document") |
        ("application/zip", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet")
    )
}

// ตรวจสอบ extension กับ MIME type
fn validate_extension_mime(filename: &str, content_type: &str) -> Result<(), String> {
    let ext = std::path::Path::new(filename)
        .extension()
        .and_then(|e| e.to_str())
        .map(|e| e.to_lowercase());
    
    let ext = match ext {
        Some(e) => e,
        None => return Ok(()),  // ไม่มี extension - ผ่าน
    };
    
    let valid = match content_type {
        "image/jpeg" => matches!(ext.as_str(), "jpg" | "jpeg"),
        "image/png" => ext == "png",
        "image/gif" => ext == "gif",
        "image/webp" => ext == "webp",
        "application/pdf" => ext == "pdf",
        "text/plain" => ext == "txt",
        "text/csv" => ext == "csv",
        _ => true,
    };
    
    if valid {
        Ok(())
    } else {
        Err(format!(
            "Extension '.{}' does not match content type '{}'",
            ext, content_type
        ))
    }
}
```

---

## Multiple File Upload

การ upload หลายไฟล์พร้อมกัน

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse, Result, Error};
use futures_util::TryStreamExt;
use std::path::PathBuf;

// Upload หลายไฟล์
async fn upload_multiple_files(
    config: web::Data<UploadConfig>,
    mut payload: Multipart,
) -> Result<HttpResponse, Error> {
    let mut results: Vec<FileUploadResult> = vec![];
    let mut errors: Vec<UploadError> = vec![];
    let mut file_count = 0;
    
    while let Some(mut field) = payload.try_next().await? {
        // ตรวจสอบ limit
        if file_count >= config.max_files_per_request {
            // อ่าน remaining fields ทิ้ง
            while field.try_next().await?.is_some() {}
            errors.push(UploadError {
                filename: "additional files".to_string(),
                error: format!("Maximum {} files per request", config.max_files_per_request),
            });
            continue;
        }
        
        let name = field.content_disposition()
            .get_name()
            .unwrap_or("")
            .to_string();
        
        // ข้าม non-file fields
        let filename = match field.content_disposition().get_filename() {
            Some(f) => sanitize_filename(f),
            None => {
                while field.try_next().await?.is_some() {}
                continue;
            }
        };
        
        if filename.is_empty() {
            while field.try_next().await?.is_some() {}
            continue;
        }
        
        file_count += 1;
        
        let content_type = field.content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_else(|| {
                // Guess from filename
                mime_guess::from_path(&filename)
                    .first_or_octet_stream()
                    .to_string()
            });
        
        // อ่านข้อมูล
        let mut data: Vec<u8> = vec![];
        let mut size_exceeded = false;
        
        while let Some(chunk) = field.try_next().await? {
            data.extend_from_slice(&chunk);
            if data.len() > config.max_file_size {
                size_exceeded = true;
                // drain remaining
                while field.try_next().await?.is_some() {}
                break;
            }
        }
        
        if size_exceeded {
            errors.push(UploadError {
                filename: filename.clone(),
                error: format!("File exceeds {} MB limit", config.max_file_size / 1024 / 1024),
            });
            continue;
        }
        
        // Validate
        let validator = FileValidator {
            allowed_types: config.allowed_types.iter().cloned().collect(),
            max_size: config.max_file_size,
        };
        
        if let Err(err) = validator.validate(&filename, &content_type, &data) {
            errors.push(UploadError { filename: filename.clone(), error: err });
            continue;
        }
        
        // Save file
        let stored_name = generate_unique_filename(&filename);
        match save_uploaded_file(&data, &config.upload_dir, &stored_name).await {
            Ok(_) => {
                results.push(FileUploadResult {
                    original_name: filename,
                    stored_name: stored_name.clone(),
                    path: config.upload_dir.join(&stored_name).to_string_lossy().to_string(),
                    url: format!("/uploads/{}", stored_name),
                    size: data.len(),
                    content_type,
                });
            },
            Err(e) => {
                log::error!("Failed to save {}: {}", filename, e);
                errors.push(UploadError {
                    filename,
                    error: "Storage error".to_string(),
                });
            }
        }
    }
    
    let status = if results.is_empty() {
        400
    } else if !errors.is_empty() {
        207  // Multi-Status
    } else {
        200
    };
    
    let response = UploadResponse {
        success: !results.is_empty(),
        files: results,
        failed: errors,
        total: file_count,
        uploaded: 0,  // will be set below
    };
    
    Ok(HttpResponse::build(actix_web::http::StatusCode::from_u16(status).unwrap())
        .json(response))
}

// App setup สำหรับ file upload
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    // สร้าง upload directories
    for dir in &["images", "documents", "other"] {
        tokio::fs::create_dir_all(format!("storage/uploads/{}", dir)).await.ok();
    }
    
    let image_config = web::Data::new(UploadConfig::for_images());
    let doc_config = web::Data::new(UploadConfig::for_documents());
    
    HttpServer::new(move || {
        App::new()
            .app_data(image_config.clone())
            .app_data(doc_config.clone())
            // ตั้ง payload limit สูงสุด
            .app_data(
                web::PayloadConfig::new(100 * 1024 * 1024)  // 100MB global limit
            )
            .service(
                web::scope("/upload")
                    .route("/image", web::post().to(upload_single_image))
                    .route("/images", web::post().to(upload_multiple_files))
                    .route("/document", web::post().to(upload_document))
                    .route("/with-metadata", web::post().to(upload_with_metadata))
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

1. **Form handling** - การรับข้อมูลจาก HTML forms (urlencoded)
2. **Multipart** - การทำงานกับ multipart/form-data
3. **actix-multipart** - Crate สำหรับ parse multipart data
4. **File upload** - การจัดการ file uploads อย่างสมบูรณ์
5. **Saving files** - การบันทึกไฟล์ลง disk อย่างปลอดภัย
6. **Size validation** - การตรวจสอบขนาดไฟล์
7. **MIME validation** - การตรวจสอบประเภทไฟล์ด้วย magic bytes
8. **Multiple uploads** - การ upload หลายไฟล์พร้อมกัน

---

## การนำทาง

- [← Part 028: CORS and Security Headers](../part_028/README.md)
- [→ Part 030: Query Builder and Pagination](../part_030/README.md)
- [กลับหน้าหลัก](../../README.md)
