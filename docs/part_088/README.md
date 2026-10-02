# Part 088: Project: File Storage Service 🗄️

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง S3-compatible File Storage Interface
- จัดเก็บ File Metadata ในฐานข้อมูล
- ทำ Access Control ต่อ File
- สร้าง Presigned URLs
- ทำ File Versioning
- สร้าง Thumbnail Generation
- ทำ Virus Scanning Integration
- สร้าง Complete File Service API

---

## 1. โครงสร้างโปรเจกต์

```
file_storage/
├── Cargo.toml
├── .env
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── file.rs
    │   └── bucket.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── buckets.rs
    │   ├── files.rs
    │   └── presigned.rs
    ├── services/
    │   ├── mod.rs
    │   ├── storage.rs
    │   ├── thumbnail.rs
    │   ├── virus_scan.rs
    │   └── presigned.rs
    └── middleware/
        ├── mod.rs
        └── auth.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "file_storage"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
actix-multipart = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
aws-sdk-s3 = "1"
aws-config = "1"
aws-credential-types = "1"
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
thiserror = "1"
futures-util = "0.3"
sha2 = "0.10"
hex = "0.4"
mime = "0.3"
image = { version = "0.25", features = ["png", "jpeg", "webp"] }
reqwest = { version = "0.12", features = ["json", "multipart"] }
rand = "0.8"
base64 = "0.22"
hmac = "0.12"
jsonwebtoken = "9"
validator = { version = "0.18", features = ["derive"] }
bytes = "1"
tokio-util = { version = "0.7", features = ["io"] }
```

---

## 3. Models

### 3.1 File Model (`src/models/file.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "file_access", rename_all = "lowercase")]
pub enum FileAccess {
    Private,
    Public,
    Authenticated,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "scan_status", rename_all = "lowercase")]
pub enum ScanStatus {
    Pending,
    Clean,
    Infected,
    Failed,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct FileRecord {
    pub id: Uuid,
    pub bucket_id: Uuid,
    pub key: String,
    pub filename: String,
    pub original_name: String,
    pub mime_type: String,
    pub size_bytes: i64,
    pub etag: String,
    pub version_id: Option<String>,
    pub access: FileAccess,
    pub owner_id: Uuid,
    pub is_current: bool,
    pub parent_id: Option<Uuid>,
    pub metadata: Option<serde_json::Value>,
    pub scan_status: ScanStatus,
    pub thumbnail_key: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct FileResponse {
    pub id: Uuid,
    pub key: String,
    pub filename: String,
    pub mime_type: String,
    pub size_bytes: i64,
    pub access: FileAccess,
    pub url: Option<String>,
    pub thumbnail_url: Option<String>,
    pub version_id: Option<String>,
    pub versions: Vec<FileVersion>,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct FileVersion {
    pub id: Uuid,
    pub version_id: Option<String>,
    pub size_bytes: i64,
    pub etag: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct FileQuery {
    pub prefix: Option<String>,
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub mime_type: Option<String>,
    pub access: Option<FileAccess>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateFileRequest {
    pub access: Option<FileAccess>,
    pub metadata: Option<serde_json::Value>,
}
```

### 3.2 Bucket Model (`src/models/bucket.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Bucket {
    pub id: Uuid,
    pub name: String,
    pub region: String,
    pub is_public: bool,
    pub max_file_size: i64,
    pub allowed_types: Vec<String>,
    pub owner_id: Uuid,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateBucketRequest {
    pub name: String,
    pub region: Option<String>,
    pub is_public: Option<bool>,
    pub max_file_size: Option<i64>,
    pub allowed_types: Option<Vec<String>>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct PresignedUrlRequest {
    pub key: String,
    pub expires_in_seconds: Option<u64>,
    pub content_type: Option<String>,
    pub max_size_bytes: Option<u64>,
}

#[derive(Debug, Serialize)]
pub struct PresignedUploadResponse {
    pub upload_url: String,
    pub fields: serde_json::Value,
    pub key: String,
    pub expires_at: DateTime<Utc>,
}
```

---

## 4. Storage Service (`src/services/storage.rs`)

```rust
use aws_sdk_s3::{
    config::{Credentials, Region},
    operation::{
        delete_object::DeleteObjectOutput,
        get_object::GetObjectOutput,
    },
    presigning::PresigningConfig,
    Client,
};
use bytes::Bytes;
use sha2::{Digest, Sha256};
use std::time::Duration;
use uuid::Uuid;

use crate::errors::AppError;

pub struct StorageService {
    client: Client,
    bucket_name: String,
    endpoint_url: Option<String>,
}

impl StorageService {
    pub async fn new(
        access_key: &str,
        secret_key: &str,
        region: &str,
        bucket_name: &str,
        endpoint_url: Option<String>,
    ) -> Self {
        let credentials = Credentials::new(
            access_key,
            secret_key,
            None,
            None,
            "file_storage_service",
        );

        let config_builder = aws_config::from_env()
            .credentials_provider(credentials)
            .region(Region::new(region.to_string()));

        let config = if let Some(endpoint) = &endpoint_url {
            config_builder
                .endpoint_url(endpoint)
                .load()
                .await
        } else {
            config_builder.load().await
        };

        let client = Client::new(&config);

        StorageService {
            client,
            bucket_name: bucket_name.to_string(),
            endpoint_url,
        }
    }

    pub async fn upload(
        &self,
        key: &str,
        data: Bytes,
        content_type: &str,
        access: &str,
    ) -> Result<UploadResult, AppError> {
        // Calculate ETag (MD5-like for compatibility)
        let etag = format!("{:x}", Sha256::digest(&data));

        let mut request = self.client
            .put_object()
            .bucket(&self.bucket_name)
            .key(key)
            .body(aws_sdk_s3::primitives::ByteStream::from(data.clone()))
            .content_type(content_type);

        if access == "public" {
            request = request.acl(aws_sdk_s3::types::ObjectCannedAcl::PublicRead);
        }

        request.send()
            .await
            .map_err(|e| AppError::InternalError(format!("S3 upload error: {}", e)))?;

        let size_bytes = data.len() as i64;
        let url = if access == "public" {
            Some(self.get_public_url(key))
        } else {
            None
        };

        Ok(UploadResult {
            key: key.to_string(),
            etag,
            size_bytes,
            url,
        })
    }

    pub async fn download(&self, key: &str) -> Result<Bytes, AppError> {
        let output = self.client
            .get_object()
            .bucket(&self.bucket_name)
            .key(key)
            .send()
            .await
            .map_err(|e| AppError::NotFound(format!("File not found: {}", e)))?;

        let data = output.body
            .collect()
            .await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        Ok(data.into_bytes())
    }

    pub async fn delete(&self, key: &str) -> Result<(), AppError> {
        self.client
            .delete_object()
            .bucket(&self.bucket_name)
            .key(key)
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("S3 delete error: {}", e)))?;

        Ok(())
    }

    pub async fn create_presigned_download_url(
        &self,
        key: &str,
        expires_in: Duration,
    ) -> Result<String, AppError> {
        let presigning_config = PresigningConfig::expires_in(expires_in)
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let presigned = self.client
            .get_object()
            .bucket(&self.bucket_name)
            .key(key)
            .presigned(presigning_config)
            .await
            .map_err(|e| AppError::InternalError(format!("Presign error: {}", e)))?;

        Ok(presigned.uri().to_string())
    }

    pub async fn create_presigned_upload_url(
        &self,
        key: &str,
        content_type: &str,
        expires_in: Duration,
    ) -> Result<String, AppError> {
        let presigning_config = PresigningConfig::expires_in(expires_in)
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let presigned = self.client
            .put_object()
            .bucket(&self.bucket_name)
            .key(key)
            .content_type(content_type)
            .presigned(presigning_config)
            .await
            .map_err(|e| AppError::InternalError(format!("Presign error: {}", e)))?;

        Ok(presigned.uri().to_string())
    }

    pub fn get_public_url(&self, key: &str) -> String {
        if let Some(endpoint) = &self.endpoint_url {
            format!("{}/{}/{}", endpoint, self.bucket_name, key)
        } else {
            format!(
                "https://{}.s3.amazonaws.com/{}",
                self.bucket_name, key
            )
        }
    }

    pub async fn copy_object(&self, source_key: &str, dest_key: &str) -> Result<(), AppError> {
        let source = format!("{}/{}", self.bucket_name, source_key);
        self.client
            .copy_object()
            .bucket(&self.bucket_name)
            .copy_source(source)
            .key(dest_key)
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("Copy error: {}", e)))?;

        Ok(())
    }
}

#[derive(Debug)]
pub struct UploadResult {
    pub key: String,
    pub etag: String,
    pub size_bytes: i64,
    pub url: Option<String>,
}
```

---

## 5. Thumbnail Service (`src/services/thumbnail.rs`)

```rust
use bytes::Bytes;
use image::{DynamicImage, ImageOutputFormat};
use std::io::Cursor;

use crate::errors::AppError;

pub struct ThumbnailService;

impl ThumbnailService {
    pub fn generate(data: &Bytes, mime_type: &str, max_width: u32, max_height: u32) -> Result<Bytes, AppError> {
        if !mime_type.starts_with("image/") {
            return Err(AppError::BadRequest("Not an image file".to_string()));
        }

        let img = image::load_from_memory(data)
            .map_err(|e| AppError::InternalError(format!("Image load error: {}", e)))?;

        // Resize while maintaining aspect ratio
        let thumbnail = img.thumbnail(max_width, max_height);

        let mut output = Cursor::new(Vec::new());
        
        let format = match mime_type {
            "image/jpeg" | "image/jpg" => ImageOutputFormat::Jpeg(85),
            "image/png" => ImageOutputFormat::Png,
            "image/webp" => ImageOutputFormat::WebP,
            _ => ImageOutputFormat::Jpeg(85),
        };

        thumbnail
            .write_to(&mut output, format)
            .map_err(|e| AppError::InternalError(format!("Thumbnail write error: {}", e)))?;

        Ok(Bytes::from(output.into_inner()))
    }

    pub fn is_image(mime_type: &str) -> bool {
        matches!(mime_type, "image/jpeg" | "image/jpg" | "image/png" | "image/gif" | "image/webp" | "image/svg+xml")
    }

    pub fn get_thumbnail_key(original_key: &str, width: u32, height: u32) -> String {
        let parts: Vec<&str> = original_key.rsplitn(2, '.').collect();
        if parts.len() == 2 {
            format!("{}_thumb_{}x{}.{}", parts[1], width, height, parts[0])
        } else {
            format!("{}_thumb_{}x{}", original_key, width, height)
        }
    }
}
```

---

## 6. Virus Scan Service (`src/services/virus_scan.rs`)

```rust
use reqwest::Client;
use bytes::Bytes;

use crate::errors::AppError;

pub struct VirusScanService {
    clamav_url: Option<String>,
    client: Client,
}

impl VirusScanService {
    pub fn new(clamav_url: Option<String>) -> Self {
        VirusScanService {
            clamav_url,
            client: Client::new(),
        }
    }

    pub async fn scan(&self, data: &Bytes, filename: &str) -> Result<ScanResult, AppError> {
        // If no scanner configured, return clean
        let url = match &self.clamav_url {
            Some(u) => u.clone(),
            None => {
                log::warn!("No virus scanner configured, skipping scan for {}", filename);
                return Ok(ScanResult {
                    is_clean: true,
                    threat_name: None,
                    scan_time_ms: 0,
                });
            }
        };

        let start = std::time::Instant::now();

        // Mock ClamAV REST API call
        // In production, use ClamAV's clamd protocol or a REST wrapper
        let form = reqwest::multipart::Form::new()
            .part("file", reqwest::multipart::Part::bytes(data.to_vec())
                .file_name(filename.to_string()));

        let response = self.client
            .post(format!("{}/scan", url))
            .multipart(form)
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("Scanner error: {}", e)))?;

        let result: serde_json::Value = response.json().await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let is_clean = result["is_clean"].as_bool().unwrap_or(true);
        let threat_name = result["threat"].as_str().map(|s| s.to_string());
        let scan_time_ms = start.elapsed().as_millis() as u64;

        Ok(ScanResult { is_clean, threat_name, scan_time_ms })
    }

    pub fn is_potentially_dangerous(mime_type: &str, filename: &str) -> bool {
        let dangerous_extensions = [
            ".exe", ".bat", ".cmd", ".sh", ".ps1", ".vbs",
            ".js", ".php", ".py", ".rb", ".pl"
        ];

        let dangerous_mimes = [
            "application/x-executable",
            "application/x-msdownload",
            "application/x-sh",
        ];

        let filename_lower = filename.to_lowercase();
        let is_dangerous_ext = dangerous_extensions.iter()
            .any(|ext| filename_lower.ends_with(ext));

        let is_dangerous_mime = dangerous_mimes.contains(&mime_type);

        is_dangerous_ext || is_dangerous_mime
    }
}

#[derive(Debug)]
pub struct ScanResult {
    pub is_clean: bool,
    pub threat_name: Option<String>,
    pub scan_time_ms: u64,
}
```

---

## 7. File Upload Handler (`src/handlers/files.rs`)

```rust
use actix_multipart::Multipart;
use actix_web::{web, HttpRequest, HttpResponse};
use bytes::Bytes;
use futures_util::StreamExt;
use sqlx::PgPool;
use std::sync::Arc;
use uuid::Uuid;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::file::{FileAccess, FileQuery, ScanStatus, UpdateFileRequest};
use crate::services::{
    storage::StorageService,
    thumbnail::ThumbnailService,
    virus_scan::VirusScanService,
};

pub async fn upload_file(
    pool: web::Data<PgPool>,
    storage: web::Data<Arc<StorageService>>,
    scanner: web::Data<Arc<VirusScanService>>,
    req: HttpRequest,
    mut multipart: Multipart,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let bucket_id = path.into_inner();

    // Verify bucket exists and user has access
    let bucket = sqlx::query!(
        "SELECT id, name, max_file_size, allowed_types, is_public FROM buckets WHERE id = $1",
        bucket_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Bucket not found".to_string()))?;

    let mut uploaded_files = Vec::new();

    while let Some(field_result) = multipart.next().await {
        let mut field = field_result
            .map_err(|e| AppError::BadRequest(format!("Multipart error: {}", e)))?;

        let content_disposition = field.content_disposition();
        let original_name = content_disposition
            .get_filename()
            .unwrap_or("unnamed")
            .to_string();

        let content_type = field
            .content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_else(|| "application/octet-stream".to_string());

        // Collect file data
        let mut data = Vec::new();
        while let Some(chunk) = field.next().await {
            let chunk = chunk.map_err(|e| AppError::InternalError(e.to_string()))?;
            data.extend_from_slice(&chunk);

            // Check max size
            if data.len() > bucket.max_file_size as usize {
                return Err(AppError::BadRequest(format!(
                    "File too large. Max size: {} bytes", bucket.max_file_size
                )));
            }
        }

        let data = Bytes::from(data);
        let file_id = Uuid::new_v4();
        let key = format!("{}/{}/{}", bucket_id, claims.sub, file_id);

        // Determine access
        let access = if bucket.is_public { FileAccess::Public } else { FileAccess::Private };

        // Virus scan
        let scan_result = scanner.scan(&data, &original_name).await?;
        if !scan_result.is_clean {
            return Err(AppError::BadRequest(format!(
                "File {} appears to be malicious: {:?}",
                original_name, scan_result.threat_name
            )));
        }

        // Upload to S3
        let upload_result = storage.upload(
            &key,
            data.clone(),
            &content_type,
            match &access {
                FileAccess::Public => "public",
                _ => "private",
            }
        ).await?;

        // Generate thumbnail if image
        let thumbnail_key = if ThumbnailService::is_image(&content_type) {
            match ThumbnailService::generate(&data, &content_type, 300, 300) {
                Ok(thumb_data) => {
                    let thumb_key = ThumbnailService::get_thumbnail_key(&key, 300, 300);
                    if storage.upload(&thumb_key, thumb_data, &content_type, "public").await.is_ok() {
                        Some(thumb_key)
                    } else {
                        None
                    }
                }
                Err(e) => {
                    log::warn!("Thumbnail generation failed: {}", e);
                    None
                }
            }
        } else {
            None
        };

        // Save metadata to database
        sqlx::query!(
            r#"
            INSERT INTO files (id, bucket_id, key, filename, original_name, mime_type,
                              size_bytes, etag, access, owner_id, scan_status, thumbnail_key)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
            "#,
            file_id,
            bucket_id,
            key,
            key.split('/').last().unwrap_or(&key),
            original_name,
            content_type,
            upload_result.size_bytes,
            upload_result.etag,
            access as FileAccess,
            claims.sub,
            ScanStatus::Clean as ScanStatus,
            thumbnail_key
        )
        .execute(pool.get_ref())
        .await?;

        uploaded_files.push(serde_json::json!({
            "id": file_id,
            "key": key,
            "filename": original_name,
            "mime_type": content_type,
            "size_bytes": upload_result.size_bytes,
            "url": upload_result.url,
            "thumbnail_key": thumbnail_key,
        }));
    }

    Ok(HttpResponse::Created().json(serde_json::json!({
        "files": uploaded_files,
        "count": uploaded_files.len(),
    })))
}

pub async fn download_file(
    pool: web::Data<PgPool>,
    storage: web::Data<Arc<StorageService>>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let file_id = path.into_inner();
    let claims = crate::middleware::auth::get_claims(&req);

    let file = sqlx::query!(
        r#"
        SELECT id, key, filename, mime_type, access::text as access, owner_id
        FROM files WHERE id = $1 AND is_current = true
        "#,
        file_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("File not found".to_string()))?;

    // Check access
    match file.access.as_deref() {
        Some("private") => {
            let user_id = claims.as_ref().map(|c| c.sub)
                .ok_or_else(|| AppError::Unauthorized("Authentication required".to_string()))?;
            if user_id != file.owner_id {
                return Err(AppError::Forbidden("Access denied".to_string()));
            }
        }
        Some("authenticated") => {
            if claims.is_none() {
                return Err(AppError::Unauthorized("Authentication required".to_string()));
            }
        }
        _ => {} // Public - no restrictions
    }

    let data = storage.download(&file.key).await?;

    Ok(HttpResponse::Ok()
        .content_type(file.mime_type)
        .append_header(("Content-Disposition", format!("inline; filename=\"{}\"", file.filename)))
        .body(data))
}

pub async fn create_presigned_url(
    pool: web::Data<PgPool>,
    storage: web::Data<Arc<StorageService>>,
    req: HttpRequest,
    path: web::Path<Uuid>,
    query: web::Query<serde_json::Value>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let file_id = path.into_inner();

    let file = sqlx::query!(
        "SELECT key, owner_id, access::text FROM files WHERE id = $1 AND is_current = true",
        file_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("File not found".to_string()))?;

    // Check ownership for private files
    if file.access.as_deref() == Some("private") && file.owner_id != claims.sub {
        return Err(AppError::Forbidden("Access denied".to_string()));
    }

    let expires_in_seconds = query.get("expires_in")
        .and_then(|v| v.as_u64())
        .unwrap_or(3600);

    let expires_in = std::time::Duration::from_secs(expires_in_seconds);
    let presigned_url = storage.create_presigned_download_url(&file.key, expires_in).await?;

    let expires_at = chrono::Utc::now() + chrono::Duration::seconds(expires_in_seconds as i64);

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "url": presigned_url,
        "expires_at": expires_at,
        "file_id": file_id,
    })))
}

pub async fn create_presigned_upload_url(
    pool: web::Data<PgPool>,
    storage: web::Data<Arc<StorageService>>,
    req: HttpRequest,
    body: web::Json<crate::models::bucket::PresignedUrlRequest>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;

    let key = format!("{}/{}/{}", "uploads", claims.sub, body.key);
    let content_type = body.content_type.as_deref().unwrap_or("application/octet-stream");
    let expires_in_seconds = body.expires_in_seconds.unwrap_or(900); // 15 minutes

    let presigned_url = storage.create_presigned_upload_url(
        &key,
        content_type,
        std::time::Duration::from_secs(expires_in_seconds)
    ).await?;

    let expires_at = chrono::Utc::now() + chrono::Duration::seconds(expires_in_seconds as i64);

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "upload_url": presigned_url,
        "key": key,
        "expires_at": expires_at,
    })))
}

pub async fn list_file_versions(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;
    let file_id = path.into_inner();

    // Get the current file to find its key
    let current = sqlx::query!(
        "SELECT key, owner_id FROM files WHERE id = $1 AND is_current = true",
        file_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("File not found".to_string()))?;

    // Get all versions
    let versions = sqlx::query!(
        r#"
        SELECT id, version_id, size_bytes, etag, is_current, created_at
        FROM files
        WHERE owner_id = $1 AND (id = $2 OR parent_id = $2)
        ORDER BY created_at DESC
        "#,
        current.owner_id, file_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(versions))
}
```

---

## 8. Main Application (`src/main.rs`)

```rust
use actix_cors::Cors;
use actix_web::{middleware::Logger, web, App, HttpServer};
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;
use std::sync::Arc;

mod errors;
mod handlers;
mod middleware;
mod models;
mod services;

use services::{
    storage::StorageService,
    virus_scan::VirusScanService,
};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::new().default_filter_or("info"));

    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let host = env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());
    let port = env::var("PORT").unwrap_or_else(|_| "8080".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(15)
        .connect(&database_url)
        .await
        .expect("Failed to create pool");

    let storage = Arc::new(StorageService::new(
        &env::var("AWS_ACCESS_KEY_ID").unwrap_or_else(|_| "minioadmin".to_string()),
        &env::var("AWS_SECRET_ACCESS_KEY").unwrap_or_else(|_| "minioadmin".to_string()),
        &env::var("AWS_REGION").unwrap_or_else(|_| "us-east-1".to_string()),
        &env::var("S3_BUCKET").expect("S3_BUCKET must be set"),
        env::var("S3_ENDPOINT").ok(),
    ).await);

    let scanner = Arc::new(VirusScanService::new(
        env::var("CLAMAV_URL").ok()
    ));

    log::info!("Starting File Storage Service at http://{}:{}", host, port);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .app_data(web::Data::new(pool.clone()))
            .app_data(web::Data::new(storage.clone()))
            .app_data(web::Data::new(scanner.clone()))
            .service(
                web::scope("/api")
                    // Buckets
                    .route("/buckets", web::post().to(handlers::buckets::create_bucket))
                    .route("/buckets", web::get().to(handlers::buckets::list_buckets))
                    .route("/buckets/{id}", web::get().to(handlers::buckets::get_bucket))
                    .route("/buckets/{id}", web::delete().to(handlers::buckets::delete_bucket))
                    // File upload
                    .route("/buckets/{id}/upload", web::post().to(handlers::files::upload_file))
                    // File operations
                    .route("/files/{id}", web::get().to(handlers::files::get_file_metadata))
                    .route("/files/{id}", web::put().to(handlers::files::update_file))
                    .route("/files/{id}", web::delete().to(handlers::files::delete_file))
                    .route("/files/{id}/download", web::get().to(handlers::files::download_file))
                    .route("/files/{id}/presign", web::get().to(handlers::files::create_presigned_url))
                    .route("/files/{id}/versions", web::get().to(handlers::files::list_file_versions))
                    // Presigned upload URL
                    .route("/presign/upload", web::post().to(handlers::files::create_presigned_upload_url))
            )
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 9. Database Schema

```sql
CREATE TYPE file_access AS ENUM ('private', 'public', 'authenticated');
CREATE TYPE scan_status AS ENUM ('pending', 'clean', 'infected', 'failed');

CREATE TABLE buckets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    region VARCHAR(50) NOT NULL DEFAULT 'us-east-1',
    is_public BOOLEAN NOT NULL DEFAULT false,
    max_file_size BIGINT NOT NULL DEFAULT 104857600, -- 100MB
    allowed_types TEXT[] NOT NULL DEFAULT '{}',
    owner_id UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bucket_id UUID NOT NULL REFERENCES buckets(id),
    key TEXT NOT NULL,
    filename VARCHAR(255) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    etag VARCHAR(64) NOT NULL,
    version_id VARCHAR(100),
    access file_access NOT NULL DEFAULT 'private',
    owner_id UUID NOT NULL REFERENCES users(id),
    is_current BOOLEAN NOT NULL DEFAULT true,
    parent_id UUID REFERENCES files(id),
    metadata JSONB,
    scan_status scan_status NOT NULL DEFAULT 'pending',
    thumbnail_key TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_files_bucket ON files(bucket_id);
CREATE INDEX idx_files_owner ON files(owner_id);
CREATE INDEX idx_files_key ON files(key);
CREATE INDEX idx_files_current ON files(is_current, owner_id);

-- File access permissions
CREATE TABLE file_permissions (
    file_id UUID NOT NULL REFERENCES files(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    can_read BOOLEAN NOT NULL DEFAULT true,
    can_write BOOLEAN NOT NULL DEFAULT false,
    can_delete BOOLEAN NOT NULL DEFAULT false,
    granted_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    PRIMARY KEY (file_id, user_id)
);
```

---

## 10. MinIO Setup (Local S3)

```yaml
# docker-compose.yml
version: '3'
services:
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

volumes:
  minio_data:
```

```bash
# .env for local MinIO
AWS_ACCESS_KEY_ID=minioadmin
AWS_SECRET_ACCESS_KEY=minioadmin
AWS_REGION=us-east-1
S3_BUCKET=file-storage
S3_ENDPOINT=http://localhost:9000
```

---

## สรุป Part 088

ใน Part นี้เราได้สร้าง File Storage Service ที่สมบูรณ์ด้วย:
1. **S3-compatible Interface** ด้วย AWS SDK Rust
2. **Multipart Upload** ผ่าน Actix-web
3. **Access Control** แบบ private/public/authenticated
4. **Presigned URLs** สำหรับ upload และ download
5. **File Versioning** ด้วย parent-child relationships
6. **Thumbnail Generation** ด้วย image crate
7. **Virus Scanning** integration แบบ configurable
8. **MinIO** สำหรับ local development

ใน **Part 089** เราจะสร้าง **API Gateway** แบบ complete

---

*[← Part 087: Analytics Dashboard API](../part_087/README.md) | [Part 089: API Gateway →](../part_089/README.md)*
