# Part 053: File Streaming and Upload 📁

## 🎯 เป้าหมายของ Part นี้

- Large file download ด้วย streaming
- actix-files สำหรับ static files
- Custom file serving พร้อม headers
- Chunked upload
- Resume upload
- Progress tracking
- AWS S3 integration
- Image resizing ด้วย image crate
- สร้าง File Management API

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
actix-files = "0.6"
actix-multipart = "0.6"
tokio = { version = "1", features = ["full"] }
tokio-util = { version = "0.7", features = ["io"] }
futures-util = "0.3"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
bytes = "1"
mime = "0.3"
mime_guess = "2"
sha2 = "0.10"
hex = "0.4"
aws-sdk-s3 = "1"
aws-config = "1"
image = "0.24"
chrono = { version = "0.4", features = ["serde"] }
anyhow = "1"
```

---

## 2. Static Files ด้วย actix-files

```rust
// src/main.rs
use actix_files::Files;
use actix_web::{App, HttpServer, web};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // Serve static files จาก ./static directory
            .service(Files::new("/static", "./static").show_files_listing())

            // Serve uploads
            .service(
                Files::new("/uploads", "./uploads")
                    .use_last_modified(true)
                    .use_etag(true),
            )

            // Single page app - fallback to index.html
            .service(
                Files::new("/", "./dist")
                    .index_file("index.html")
                    .default_handler(
                        actix_files::NamedFile::open("./dist/index.html").unwrap(),
                    ),
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 3. Custom File Serving

```rust
// src/file_serve.rs
use actix_web::{web, HttpRequest, HttpResponse};
use actix_files::NamedFile;
use std::path::PathBuf;

// Serve file พร้อม custom headers
pub async fn serve_file(
    req: HttpRequest,
    path: web::Path<String>,
) -> actix_web::Result<NamedFile> {
    let filename = path.into_inner();

    // Sanitize path - ป้องกัน path traversal
    if filename.contains("..") || filename.contains('/') {
        return Err(actix_web::error::ErrorBadRequest("Invalid filename"));
    }

    let file_path = PathBuf::from("./uploads").join(&filename);
    let file = NamedFile::open(file_path)
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))?;

    // Set headers
    Ok(file
        .use_last_modified(true)
        .use_etag(true)
        .set_content_disposition(actix_web::http::header::ContentDisposition {
            disposition: actix_web::http::header::DispositionType::Attachment,
            parameters: vec![
                actix_web::http::header::DispositionParam::Filename(filename),
            ],
        }))
}

// Stream large file
pub async fn stream_large_file(
    req: HttpRequest,
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    use tokio::fs::File;
    use tokio_util::io::ReaderStream;

    let filename = path.into_inner();
    let file_path = PathBuf::from("./large_files").join(&filename);

    let file = File::open(&file_path)
        .await
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))?;

    let metadata = tokio::fs::metadata(&file_path)
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?;

    let mime_type = mime_guess::from_path(&file_path)
        .first_or_octet_stream()
        .to_string();

    // สร้าง stream จาก file
    let stream = ReaderStream::new(file);

    Ok(HttpResponse::Ok()
        .content_type(mime_type)
        .append_header(("Content-Length", metadata.len()))
        .append_header(("Accept-Ranges", "bytes"))
        .streaming(stream))
}
```

---

## 4. Range Request (Partial Content)

Range requests ช่วยให้ client ดาวน์โหลดบางส่วนของไฟล์ได้

```rust
// src/range_serve.rs
use actix_web::{web, HttpRequest, HttpResponse};
use std::path::PathBuf;
use tokio::io::{AsyncReadExt, AsyncSeekExt};
use tokio::fs::File;

pub async fn serve_with_range(
    req: HttpRequest,
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    let filename = path.into_inner();
    let file_path = PathBuf::from("./files").join(&filename);

    let metadata = tokio::fs::metadata(&file_path)
        .await
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))?;

    let file_size = metadata.len();

    // อ่าน Range header
    let range = req
        .headers()
        .get("Range")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| parse_range(v, file_size));

    let mime_type = mime_guess::from_path(&file_path)
        .first_or_octet_stream()
        .to_string();

    match range {
        Some((start, end)) => {
            // Partial content
            let length = end - start + 1;
            let mut file = File::open(&file_path)
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;

            file.seek(std::io::SeekFrom::Start(start))
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;

            let mut buffer = vec![0u8; length as usize];
            file.read_exact(&mut buffer)
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;

            Ok(HttpResponse::PartialContent()
                .content_type(mime_type)
                .append_header(("Content-Range", format!("bytes {}-{}/{}", start, end, file_size)))
                .append_header(("Content-Length", length))
                .append_header(("Accept-Ranges", "bytes"))
                .body(buffer))
        }
        None => {
            // Full file
            let file = File::open(&file_path)
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;

            let stream = tokio_util::io::ReaderStream::new(file);

            Ok(HttpResponse::Ok()
                .content_type(mime_type)
                .append_header(("Content-Length", file_size))
                .append_header(("Accept-Ranges", "bytes"))
                .streaming(stream))
        }
    }
}

fn parse_range(range: &str, file_size: u64) -> Option<(u64, u64)> {
    let range = range.strip_prefix("bytes=")?;
    let mut parts = range.splitn(2, '-');

    let start: u64 = parts.next()?.parse().ok()?;
    let end: u64 = parts
        .next()
        .and_then(|s| s.parse().ok())
        .unwrap_or(file_size - 1);

    if start > end || end >= file_size {
        return None;
    }

    Some((start, end))
}
```

---

## 5. Multipart Upload

```rust
// src/upload.rs
use actix_multipart::Multipart;
use actix_web::{web, HttpResponse};
use futures_util::TryStreamExt;
use std::path::PathBuf;
use tokio::io::AsyncWriteExt;
use uuid::Uuid;
use sha2::{Sha256, Digest};

#[derive(serde::Serialize)]
pub struct UploadResult {
    pub file_id: String,
    pub filename: String,
    pub size: usize,
    pub content_type: String,
    pub checksum: String,
    pub url: String,
}

pub async fn upload_file(
    mut payload: Multipart,
) -> actix_web::Result<HttpResponse> {
    let mut results = Vec::new();

    // Iterate ผ่าน multipart fields
    while let Ok(Some(mut field)) = payload.try_next().await {
        let content_disposition = field.content_disposition();
        let filename = content_disposition
            .get_filename()
            .map(|f| sanitize_filename(f))
            .unwrap_or_else(|| "unnamed".to_string());

        let content_type = field
            .content_type()
            .map(|ct| ct.to_string())
            .unwrap_or_else(|| "application/octet-stream".to_string());

        let file_id = Uuid::new_v4().to_string();
        let ext = std::path::Path::new(&filename)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("bin");
        let stored_name = format!("{}.{}", file_id, ext);
        let file_path = PathBuf::from("./uploads").join(&stored_name);

        // สร้าง directory ถ้าไม่มี
        tokio::fs::create_dir_all("./uploads")
            .await
            .map_err(actix_web::error::ErrorInternalServerError)?;

        let mut file = tokio::fs::File::create(&file_path)
            .await
            .map_err(actix_web::error::ErrorInternalServerError)?;

        let mut total_size = 0usize;
        let mut hasher = Sha256::new();

        // Read chunks และ write ไปยัง file
        while let Some(chunk) = field.try_next().await.map_err(|e| {
            actix_web::error::ErrorBadRequest(format!("Failed to read chunk: {}", e))
        })? {
            total_size += chunk.len();
            hasher.update(&chunk);
            file.write_all(&chunk)
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;
        }

        let checksum = hex::encode(hasher.finalize());

        results.push(UploadResult {
            file_id: file_id.clone(),
            filename: filename.clone(),
            size: total_size,
            content_type,
            checksum,
            url: format!("/files/{}", stored_name),
        });
    }

    Ok(HttpResponse::Ok().json(results))
}

fn sanitize_filename(filename: &str) -> String {
    filename
        .chars()
        .filter(|c| c.is_alphanumeric() || *c == '.' || *c == '-' || *c == '_')
        .collect()
}
```

---

## 6. Chunked Upload (Resume)

```rust
// src/chunked_upload.rs
use actix_web::{web, HttpRequest, HttpResponse};
use serde::{Deserialize, Serialize};
use std::{collections::HashMap, sync::Arc};
use tokio::{io::AsyncWriteExt, sync::RwLock};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct UploadSession {
    pub upload_id: String,
    pub filename: String,
    pub total_size: u64,
    pub chunk_size: u64,
    pub uploaded_chunks: Vec<u32>,
    pub created_at: chrono::DateTime<chrono::Utc>,
}

pub struct UploadState {
    pub sessions: Arc<RwLock<HashMap<String, UploadSession>>>,
}

#[derive(Deserialize)]
pub struct InitUploadRequest {
    pub filename: String,
    pub total_size: u64,
    pub chunk_size: u64,
}

// 1. Init upload session
pub async fn init_upload(
    state: web::Data<UploadState>,
    body: web::Json<InitUploadRequest>,
) -> HttpResponse {
    let upload_id = Uuid::new_v4().to_string();
    let total_chunks = (body.total_size + body.chunk_size - 1) / body.chunk_size;

    let session = UploadSession {
        upload_id: upload_id.clone(),
        filename: body.filename.clone(),
        total_size: body.total_size,
        chunk_size: body.chunk_size,
        uploaded_chunks: Vec::new(),
        created_at: chrono::Utc::now(),
    };

    state.sessions.write().await.insert(upload_id.clone(), session);

    // สร้าง temp file
    let temp_path = format!("./tmp/{}", upload_id);
    tokio::fs::create_dir_all("./tmp").await.ok();

    // Pre-allocate file
    let file = tokio::fs::File::create(&temp_path).await.unwrap();
    file.set_len(body.total_size).await.ok();

    HttpResponse::Ok().json(serde_json::json!({
        "upload_id": upload_id,
        "total_chunks": total_chunks,
        "chunk_size": body.chunk_size
    }))
}

// 2. Upload chunk
pub async fn upload_chunk(
    req: HttpRequest,
    state: web::Data<UploadState>,
    path: web::Path<String>,
    body: web::Bytes,
) -> actix_web::Result<HttpResponse> {
    let upload_id = path.into_inner();

    // อ่าน chunk number จาก header
    let chunk_number: u32 = req
        .headers()
        .get("X-Chunk-Number")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.parse().ok())
        .ok_or_else(|| actix_web::error::ErrorBadRequest("Missing X-Chunk-Number header"))?;

    let mut sessions = state.sessions.write().await;
    let session = sessions
        .get_mut(&upload_id)
        .ok_or_else(|| actix_web::error::ErrorNotFound("Upload session not found"))?;

    // คำนวณ offset
    let offset = chunk_number as u64 * session.chunk_size;

    // Write chunk ที่ตำแหน่งที่ถูกต้อง
    let temp_path = format!("./tmp/{}", upload_id);
    let mut file = tokio::fs::OpenOptions::new()
        .write(true)
        .open(&temp_path)
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?;

    use tokio::io::AsyncSeekExt;
    file.seek(std::io::SeekFrom::Start(offset))
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?;

    file.write_all(&body)
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?;

    session.uploaded_chunks.push(chunk_number);

    let total_chunks =
        (session.total_size + session.chunk_size - 1) / session.chunk_size;
    let is_complete = session.uploaded_chunks.len() as u64 >= total_chunks;

    if is_complete {
        // Move temp file to final location
        let final_path = format!("./uploads/{}", session.filename);
        tokio::fs::rename(&temp_path, &final_path)
            .await
            .map_err(actix_web::error::ErrorInternalServerError)?;

        return Ok(HttpResponse::Ok().json(serde_json::json!({
            "status": "complete",
            "url": format!("/files/{}", session.filename)
        })));
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "status": "partial",
        "uploaded_chunks": session.uploaded_chunks.len(),
        "total_chunks": total_chunks
    })))
}

// 3. Get upload status
pub async fn get_upload_status(
    state: web::Data<UploadState>,
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    let upload_id = path.into_inner();
    let sessions = state.sessions.read().await;

    let session = sessions
        .get(&upload_id)
        .ok_or_else(|| actix_web::error::ErrorNotFound("Upload session not found"))?;

    Ok(HttpResponse::Ok().json(session))
}
```

---

## 7. AWS S3 Integration

```rust
// src/s3.rs
use aws_sdk_s3::{Client, primitives::ByteStream};
use aws_config::meta::region::RegionProviderChain;
use std::path::Path;

pub struct S3Client {
    client: Client,
    bucket: String,
}

impl S3Client {
    pub async fn new(bucket: &str) -> Self {
        let region_provider = RegionProviderChain::default_provider()
            .or_else("us-east-1");

        let config = aws_config::defaults(aws_config::BehaviorVersion::latest())
            .region(region_provider)
            .load()
            .await;

        let client = Client::new(&config);

        Self {
            client,
            bucket: bucket.to_string(),
        }
    }

    // Upload file ไปยัง S3
    pub async fn upload_file(
        &self,
        local_path: &Path,
        s3_key: &str,
        content_type: &str,
    ) -> anyhow::Result<String> {
        let body = ByteStream::from_path(local_path).await?;

        self.client
            .put_object()
            .bucket(&self.bucket)
            .key(s3_key)
            .body(body)
            .content_type(content_type)
            .send()
            .await?;

        let url = format!(
            "https://{}.s3.amazonaws.com/{}",
            self.bucket, s3_key
        );

        Ok(url)
    }

    // Upload bytes ไปยัง S3
    pub async fn upload_bytes(
        &self,
        data: Vec<u8>,
        s3_key: &str,
        content_type: &str,
    ) -> anyhow::Result<String> {
        let body = ByteStream::from(data);

        self.client
            .put_object()
            .bucket(&self.bucket)
            .key(s3_key)
            .body(body)
            .content_type(content_type)
            .send()
            .await?;

        Ok(format!("https://{}.s3.amazonaws.com/{}", self.bucket, s3_key))
    }

    // Download จาก S3
    pub async fn download_file(&self, s3_key: &str) -> anyhow::Result<Vec<u8>> {
        let response = self.client
            .get_object()
            .bucket(&self.bucket)
            .key(s3_key)
            .send()
            .await?;

        let data = response.body.collect().await?.into_bytes().to_vec();
        Ok(data)
    }

    // Generate presigned URL
    pub async fn presigned_url(
        &self,
        s3_key: &str,
        expires_in_secs: u64,
    ) -> anyhow::Result<String> {
        use aws_sdk_s3::presigning::PresigningConfig;
        use std::time::Duration;

        let presigned = self.client
            .get_object()
            .bucket(&self.bucket)
            .key(s3_key)
            .presigned(PresigningConfig::expires_in(
                Duration::from_secs(expires_in_secs),
            )?)
            .await?;

        Ok(presigned.uri().to_string())
    }

    // Delete file
    pub async fn delete_file(&self, s3_key: &str) -> anyhow::Result<()> {
        self.client
            .delete_object()
            .bucket(&self.bucket)
            .key(s3_key)
            .send()
            .await?;

        Ok(())
    }

    // List files with prefix
    pub async fn list_files(&self, prefix: &str) -> anyhow::Result<Vec<String>> {
        let response = self.client
            .list_objects_v2()
            .bucket(&self.bucket)
            .prefix(prefix)
            .send()
            .await?;

        let keys = response
            .contents()
            .iter()
            .filter_map(|obj| obj.key())
            .map(|k| k.to_string())
            .collect();

        Ok(keys)
    }
}
```

---

## 8. Image Processing

```rust
// src/image_process.rs
use image::{DynamicImage, ImageFormat, imageops::FilterType};
use std::io::Cursor;

pub struct ImageProcessor;

impl ImageProcessor {
    // Resize image
    pub fn resize(
        data: &[u8],
        width: u32,
        height: u32,
        maintain_aspect: bool,
    ) -> anyhow::Result<Vec<u8>> {
        let img = image::load_from_memory(data)?;

        let resized = if maintain_aspect {
            img.resize(width, height, FilterType::Lanczos3)
        } else {
            img.resize_exact(width, height, FilterType::Lanczos3)
        };

        let mut output = Vec::new();
        resized.write_to(&mut Cursor::new(&mut output), ImageFormat::Jpeg)?;

        Ok(output)
    }

    // Create thumbnail
    pub fn thumbnail(data: &[u8], size: u32) -> anyhow::Result<Vec<u8>> {
        let img = image::load_from_memory(data)?;
        let thumb = img.thumbnail(size, size);

        let mut output = Vec::new();
        thumb.write_to(&mut Cursor::new(&mut output), ImageFormat::Jpeg)?;

        Ok(output)
    }

    // Convert format
    pub fn convert(
        data: &[u8],
        format: ImageFormat,
        quality: u8,
    ) -> anyhow::Result<Vec<u8>> {
        let img = image::load_from_memory(data)?;

        let mut output = Vec::new();
        let cursor = Cursor::new(&mut output);

        match format {
            ImageFormat::Jpeg => {
                use image::codecs::jpeg::JpegEncoder;
                let encoder = JpegEncoder::new_with_quality(cursor, quality);
                img.write_with_encoder(encoder)?;
            }
            ImageFormat::Png | ImageFormat::WebP => {
                img.write_to(&mut Cursor::new(&mut output), format)?;
            }
            _ => {
                img.write_to(&mut Cursor::new(&mut output), format)?;
            }
        }

        Ok(output)
    }

    // Get image dimensions
    pub fn get_dimensions(data: &[u8]) -> anyhow::Result<(u32, u32)> {
        let img = image::load_from_memory(data)?;
        Ok((img.width(), img.height()))
    }
}
```

---

## 9. Practical: File Management API

```rust
// src/api/files.rs - Complete file management
use actix_web::{web, HttpRequest, HttpResponse};
use actix_multipart::Multipart;
use futures_util::TryStreamExt;
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use std::path::PathBuf;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct FileRecord {
    pub id: String,
    pub original_name: String,
    pub stored_name: String,
    pub size: u64,
    pub content_type: String,
    pub url: String,
    pub thumbnail_url: Option<String>,
    pub uploaded_at: String,
}

pub struct FileStorage {
    base_path: PathBuf,
    max_file_size: u64,
}

impl FileStorage {
    pub fn new(base_path: &str, max_size_mb: u64) -> Self {
        Self {
            base_path: PathBuf::from(base_path),
            max_file_size: max_size_mb * 1024 * 1024,
        }
    }

    pub async fn save(
        &self,
        data: &[u8],
        filename: &str,
        content_type: &str,
    ) -> anyhow::Result<FileRecord> {
        if data.len() as u64 > self.max_file_size {
            anyhow::bail!("File too large");
        }

        let id = Uuid::new_v4().to_string();
        let ext = std::path::Path::new(filename)
            .extension()
            .and_then(|e| e.to_str())
            .unwrap_or("bin");
        let stored_name = format!("{}.{}", id, ext);

        // สร้าง subdirectory จาก 2 ตัวแรกของ ID (เหมือน git)
        let subdir = &id[..2];
        let dir = self.base_path.join(subdir);
        tokio::fs::create_dir_all(&dir).await?;

        let file_path = dir.join(&stored_name);
        tokio::fs::write(&file_path, data).await?;

        // สร้าง thumbnail ถ้าเป็นรูปภาพ
        let thumbnail_url = if content_type.starts_with("image/") {
            if let Ok(thumb) = crate::image_process::ImageProcessor::thumbnail(data, 200) {
                let thumb_name = format!("thumb_{}", stored_name);
                let thumb_path = dir.join(&thumb_name);
                tokio::fs::write(&thumb_path, &thumb).await.ok();
                Some(format!("/files/thumbs/{}/{}", subdir, thumb_name))
            } else {
                None
            }
        } else {
            None
        };

        Ok(FileRecord {
            id,
            original_name: filename.to_string(),
            stored_name: stored_name.clone(),
            size: data.len() as u64,
            content_type: content_type.to_string(),
            url: format!("/files/{}/{}", subdir, stored_name),
            thumbnail_url,
            uploaded_at: chrono::Utc::now().to_rfc3339(),
        })
    }
}

// Upload handler
pub async fn upload(
    storage: web::Data<FileStorage>,
    mut payload: Multipart,
) -> actix_web::Result<HttpResponse> {
    let mut uploaded = Vec::new();

    while let Ok(Some(mut field)) = payload.try_next().await {
        let cd = field.content_disposition();
        let filename = cd
            .get_filename()
            .map(|f| f.to_string())
            .unwrap_or_else(|| "file".to_string());

        let ct = field
            .content_type()
            .map(|c| c.to_string())
            .unwrap_or("application/octet-stream".to_string());

        // Read all bytes
        let mut data = Vec::new();
        while let Some(chunk) = field.try_next().await.map_err(|e| {
            actix_web::error::ErrorBadRequest(e.to_string())
        })? {
            data.extend_from_slice(&chunk);
        }

        let record = storage
            .save(&data, &filename, &ct)
            .await
            .map_err(actix_web::error::ErrorInternalServerError)?;

        uploaded.push(record);
    }

    Ok(HttpResponse::Ok().json(uploaded))
}

// Download handler with streaming
pub async fn download(
    path: web::Path<(String, String)>,
) -> actix_web::Result<actix_files::NamedFile> {
    let (subdir, filename) = path.into_inner();

    // Validate
    if subdir.len() != 2 || subdir.chars().any(|c| !c.is_alphanumeric()) {
        return Err(actix_web::error::ErrorBadRequest("Invalid path"));
    }

    let file_path = PathBuf::from("./uploads").join(&subdir).join(&filename);

    actix_files::NamedFile::open(file_path)
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))
}

// Delete file
pub async fn delete_file(
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    let file_id = path.into_inner();

    // หา file จาก ID
    let subdir = &file_id[..2];
    let dir = PathBuf::from("./uploads").join(subdir);

    // ลบทุก file ที่มี prefix เป็น file_id
    let mut entries = tokio::fs::read_dir(&dir)
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?;

    while let Some(entry) = entries
        .next_entry()
        .await
        .map_err(actix_web::error::ErrorInternalServerError)?
    {
        let name = entry.file_name().to_string_lossy().to_string();
        if name.starts_with(&file_id) {
            tokio::fs::remove_file(entry.path())
                .await
                .map_err(actix_web::error::ErrorInternalServerError)?;
        }
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "deleted": true,
        "file_id": file_id
    })))
}
```

---

## สรุป

✅ Static files ด้วย actix-files  
✅ Custom file serving พร้อม headers  
✅ Range requests (partial content)  
✅ Multipart file upload  
✅ Chunked resume upload  
✅ AWS S3 integration  
✅ Image resizing ด้วย image crate  
✅ Complete file management API  

### Exercise

1. เพิ่ม virus scanning ก่อน save file
2. Implement file deduplication ด้วย checksum
3. เพิ่ม CDN URL generation
4. สร้าง image transformation pipeline (resize, crop, watermark)

---

*[← Part 052: Server-Sent Events](../part_052/README.md) | [Part 054: Background Jobs with Tokio →](../part_054/README.md)*
