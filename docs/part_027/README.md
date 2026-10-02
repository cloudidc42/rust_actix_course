# Part 027: Static Files and Templates ใน Actix-web

## สารบัญ
- [แนะนำ Static Files](#แนะนำ-static-files)
- [actix-files Crate](#actix-files-crate)
- [Serving Static Directory](#serving-static-directory)
- [Single File Serving](#single-file-serving)
- [Template Engines: Tera](#template-engines-tera)
- [Tera Templates กับ Context](#tera-templates-กับ-context)
- [HTML Forms](#html-forms)
- [File Download](#file-download)
- [Content-Type Handling](#content-type-handling)

---

## แนะนำ Static Files

Web applications มักต้องการ serve static files เช่น HTML, CSS, JavaScript, images และอาจต้องการ template engine สำหรับ server-side rendering

### Dependencies

```toml
[dependencies]
actix-web = "4"
actix-files = "0.6"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
tera = "1"
mime = "0.3"
mime_guess = "2"
```

### โครงสร้าง Project

```
project/
├── src/
│   └── main.rs
├── static/
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── app.js
│   └── images/
│       └── logo.png
├── templates/
│   ├── base.html
│   ├── index.html
│   ├── users/
│   │   ├── list.html
│   │   └── detail.html
│   └── forms/
│       └── contact.html
└── Cargo.toml
```

---

## actix-files Crate

`actix-files` ให้ utilities สำหรับ serve static files

```rust
use actix_files as fs;
use actix_web::{web, App, HttpServer, HttpResponse, middleware};

// Basic setup
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            // Serve ไฟล์ทั้ง directory "/static" ที่ path "/static"
            .service(
                fs::Files::new("/static", "static")
                    .show_files_listing()  // แสดง directory listing
                    .index_file("index.html")  // default file
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## Serving Static Directory

การ serve static files จาก directory

```rust
use actix_files as fs;
use actix_web::{web, App, HttpServer, HttpResponse};
use std::path::PathBuf;

// Serve static files with configuration
fn configure_static_files(cfg: &mut web::ServiceConfig) {
    cfg
        // Public static files - ไม่ต้อง auth
        .service(
            fs::Files::new("/public", "static/public")
                .index_file("index.html")
                .prefer_utf8(true)  // ใช้ UTF-8 content type
        )
        // Assets - CSS, JS, images
        .service(
            fs::Files::new("/assets", "static/assets")
                .use_etag(true)          // ใช้ ETag caching
                .use_last_modified(true)  // ใช้ Last-Modified header
        )
        // Uploads - user uploaded files
        .service(
            fs::Files::new("/uploads", "storage/uploads")
                .index_file("index.html")
                // ไม่แสดง directory listing สำหรับ uploads
                .disable_content_disposition()
        );
}

// App ที่ configure static files
async fn start_app_with_static() -> std::io::Result<()> {
    // สร้าง directories ถ้ายังไม่มี
    std::fs::create_dir_all("static/public").ok();
    std::fs::create_dir_all("static/assets/css").ok();
    std::fs::create_dir_all("static/assets/js").ok();
    std::fs::create_dir_all("static/assets/images").ok();
    std::fs::create_dir_all("storage/uploads").ok();
    
    HttpServer::new(|| {
        App::new()
            .configure(configure_static_files)
            .route("/api/status", web::get().to(|| async {
                HttpResponse::Ok().json(serde_json::json!({"status": "ok"}))
            }))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// Serve static files พร้อม custom headers
use actix_web::dev::{ServiceRequest, ServiceResponse};
use actix_web::Error;

async fn serve_with_cache_headers() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(
                fs::Files::new("/assets", "static/assets")
                    // เพิ่ม cache headers
                    .use_etag(true)
                    .use_last_modified(true)
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

// ตัวอย่างการสร้าง static files สำหรับ demo
fn create_demo_files() {
    // สร้าง HTML file
    let html = r#"<!DOCTYPE html>
<html>
<head>
    <title>Demo</title>
    <link rel="stylesheet" href="/assets/css/style.css">
</head>
<body>
    <h1>Hello from static file!</h1>
    <script src="/assets/js/app.js"></script>
</body>
</html>"#;
    
    std::fs::write("static/public/index.html", html).ok();
    
    // สร้าง CSS file
    let css = r#"
body {
    font-family: Arial, sans-serif;
    margin: 0;
    padding: 20px;
    background: #f0f0f0;
}
h1 {
    color: #333;
}
"#;
    std::fs::create_dir_all("static/assets/css").ok();
    std::fs::write("static/assets/css/style.css", css).ok();
    
    // สร้าง JS file
    let js = r#"
console.log('App loaded');

document.addEventListener('DOMContentLoaded', function() {
    console.log('DOM ready');
});
"#;
    std::fs::create_dir_all("static/assets/js").ok();
    std::fs::write("static/assets/js/app.js", js).ok();
}
```

---

## Single File Serving

การ serve ไฟล์เดี่ยว

```rust
use actix_files::NamedFile;
use actix_web::{web, HttpRequest, HttpResponse, Result};
use std::path::PathBuf;

// Serve single file
async fn serve_favicon() -> Result<NamedFile> {
    Ok(NamedFile::open("static/favicon.ico")?)
}

// Serve file ตาม path parameter
async fn serve_file(path: web::Path<PathBuf>) -> Result<NamedFile> {
    let file_path = PathBuf::from("static").join(path.into_inner());
    
    // Security: ป้องกัน path traversal
    let canonical = file_path.canonicalize()?;
    let static_dir = PathBuf::from("static").canonicalize()?;
    
    if !canonical.starts_with(&static_dir) {
        return Err(actix_web::error::ErrorForbidden("Access denied"));
    }
    
    Ok(NamedFile::open(canonical)?)
}

// Serve file พร้อม custom headers
async fn serve_file_with_headers(
    req: HttpRequest,
    path: web::Path<String>,
) -> Result<HttpResponse> {
    let file_name = path.into_inner();
    let file_path = format!("static/{}", file_name);
    
    let file = std::fs::read(&file_path)
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))?;
    
    // Guess content type
    let mime_type = mime_guess::from_path(&file_path)
        .first_or_octet_stream();
    
    Ok(HttpResponse::Ok()
        .content_type(mime_type.to_string())
        .append_header(("Cache-Control", "public, max-age=86400"))
        .append_header(("X-Content-Type-Options", "nosniff"))
        .body(file))
}

// Download specific file
async fn download_file(path: web::Path<String>) -> Result<NamedFile> {
    let file_name = path.into_inner();
    let file_path = format!("storage/downloads/{}", file_name);
    
    let file = NamedFile::open(file_path)?
        .set_content_disposition(actix_web::http::header::ContentDisposition {
            disposition: actix_web::http::header::DispositionType::Attachment,
            parameters: vec![
                actix_web::http::header::DispositionParam::Filename(file_name)
            ],
        });
    
    Ok(file)
}
```

---

## Template Engines: Tera

Tera คือ template engine สำหรับ Rust ที่ได้รับแรงบันดาลใจจาก Jinja2/Django

### การตั้งค่า Tera

```rust
use tera::Tera;
use actix_web::{web, App, HttpServer};
use std::sync::Arc;

// Initialize Tera
fn create_tera() -> Tera {
    match Tera::new("templates/**/*") {
        Ok(t) => t,
        Err(e) => {
            println!("Parsing error(s): {}", e);
            std::process::exit(1);
        }
    }
}

// App setup กับ Tera
async fn setup_with_tera() -> std::io::Result<()> {
    // สร้าง templates directory
    std::fs::create_dir_all("templates/users").ok();
    
    // สร้าง templates
    create_base_template();
    create_index_template();
    create_users_template();
    
    let tera = web::Data::new(create_tera());
    
    HttpServer::new(move || {
        App::new()
            .app_data(tera.clone())
            .route("/", web::get().to(index_page))
            .route("/users", web::get().to(users_page))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

fn create_base_template() {
    let base_html = r#"<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}My App{% endblock %}</title>
    <link rel="stylesheet" href="/assets/css/style.css">
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: 'Sarabun', Arial, sans-serif; background: #f5f5f5; }
        nav { background: #333; padding: 1rem; }
        nav a { color: white; text-decoration: none; margin-right: 1rem; }
        nav a:hover { text-decoration: underline; }
        main { max-width: 1200px; margin: 2rem auto; padding: 0 1rem; }
        footer { text-align: center; padding: 2rem; color: #666; }
        .btn { padding: 0.5rem 1rem; border: none; cursor: pointer; border-radius: 4px; }
        .btn-primary { background: #007bff; color: white; }
        .btn-danger { background: #dc3545; color: white; }
        .card { background: white; padding: 1.5rem; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); margin-bottom: 1rem; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 0.75rem; border-bottom: 1px solid #ddd; text-align: left; }
        th { background: #f8f9fa; font-weight: bold; }
        tr:hover { background: #f5f5f5; }
        .alert { padding: 1rem; border-radius: 4px; margin-bottom: 1rem; }
        .alert-success { background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .alert-error { background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
        form .form-group { margin-bottom: 1rem; }
        form label { display: block; margin-bottom: 0.25rem; font-weight: bold; }
        form input, form textarea, form select { 
            width: 100%; padding: 0.5rem; border: 1px solid #ddd; 
            border-radius: 4px; font-size: 1rem; 
        }
        form .error { color: #dc3545; font-size: 0.875rem; margin-top: 0.25rem; }
    </style>
</head>
<body>
    <nav>
        <a href="/">หน้าหลัก</a>
        <a href="/users">ผู้ใช้งาน</a>
        <a href="/contact">ติดต่อ</a>
    </nav>
    <main>
        {% if flash_message %}
        <div class="alert alert-{{ flash_type | default(value='success') }}">
            {{ flash_message }}
        </div>
        {% endif %}
        
        {% block content %}{% endblock %}
    </main>
    <footer>
        <p>&copy; 2024 My App. All rights reserved.</p>
    </footer>
    <script src="/assets/js/app.js"></script>
    {% block scripts %}{% endblock %}
</body>
</html>"#;
    
    std::fs::write("templates/base.html", base_html).ok();
}

fn create_index_template() {
    let index_html = r#"{% extends "base.html" %}

{% block title %}หน้าหลัก - My App{% endblock %}

{% block content %}
<div class="card">
    <h1>ยินดีต้อนรับสู่ {{ app_name }}</h1>
    <p>{{ description }}</p>
    <p>วันที่: {{ current_date }}</p>
</div>

<div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 1rem; margin-top: 1rem;">
    <div class="card">
        <h3>ผู้ใช้ทั้งหมด</h3>
        <p style="font-size: 2rem; font-weight: bold; color: #007bff;">{{ stats.users }}</p>
    </div>
    <div class="card">
        <h3>โพสต์ทั้งหมด</h3>
        <p style="font-size: 2rem; font-weight: bold; color: #28a745;">{{ stats.posts }}</p>
    </div>
    <div class="card">
        <h3>ความคิดเห็น</h3>
        <p style="font-size: 2rem; font-weight: bold; color: #ffc107;">{{ stats.comments }}</p>
    </div>
</div>

{% if featured_items %}
<div class="card" style="margin-top: 1rem;">
    <h2>รายการแนะนำ</h2>
    <ul>
    {% for item in featured_items %}
        <li>
            <strong>{{ item.title }}</strong>
            {% if item.is_new %}
                <span style="color: red; font-size: 0.8rem;">[ใหม่]</span>
            {% endif %}
            - {{ item.description }}
        </li>
    {% endfor %}
    </ul>
</div>
{% endif %}
{% endblock %}"#;
    
    std::fs::write("templates/index.html", index_html).ok();
}

fn create_users_template() {
    std::fs::create_dir_all("templates/users").ok();
    
    let users_list_html = r#"{% extends "base.html" %}

{% block title %}รายการผู้ใช้ - My App{% endblock %}

{% block content %}
<div class="card">
    <div style="display: flex; justify-content: space-between; align-items: center;">
        <h1>ผู้ใช้งานทั้งหมด ({{ users | length }} คน)</h1>
        <a href="/users/new" class="btn btn-primary">เพิ่มผู้ใช้ใหม่</a>
    </div>
    
    {% if users %}
    <table style="margin-top: 1rem;">
        <thead>
            <tr>
                <th>#</th>
                <th>ชื่อผู้ใช้</th>
                <th>อีเมล</th>
                <th>สถานะ</th>
                <th>วันที่สร้าง</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {% for user in users %}
            <tr>
                <td>{{ loop.index }}</td>
                <td>{{ user.username }}</td>
                <td>{{ user.email }}</td>
                <td>
                    {% if user.is_active %}
                        <span style="color: green;">✓ Active</span>
                    {% else %}
                        <span style="color: red;">✗ Inactive</span>
                    {% endif %}
                </td>
                <td>{{ user.created_at }}</td>
                <td>
                    <a href="/users/{{ user.id }}" style="color: #007bff;">ดูรายละเอียด</a>
                    &nbsp;|&nbsp;
                    <a href="/users/{{ user.id }}/edit" style="color: #28a745;">แก้ไข</a>
                </td>
            </tr>
            {% endfor %}
        </tbody>
    </table>
    
    <!-- Pagination -->
    {% if pagination.total_pages > 1 %}
    <div style="display: flex; justify-content: center; gap: 0.5rem; margin-top: 1rem;">
        {% if pagination.has_prev %}
            <a href="?page={{ pagination.prev_page }}" class="btn btn-primary">← ก่อนหน้า</a>
        {% endif %}
        
        <span style="padding: 0.5rem 1rem;">
            หน้า {{ pagination.page }} จาก {{ pagination.total_pages }}
        </span>
        
        {% if pagination.has_next %}
            <a href="?page={{ pagination.next_page }}" class="btn btn-primary">ถัดไป →</a>
        {% endif %}
    </div>
    {% endif %}
    
    {% else %}
    <p style="text-align: center; color: #666; margin-top: 2rem;">ยังไม่มีผู้ใช้งาน</p>
    {% endif %}
</div>
{% endblock %}"#;
    
    std::fs::write("templates/users/list.html", users_list_html).ok();
}
```

---

## Tera Templates กับ Context

การส่งข้อมูลไปยัง templates

```rust
use tera::{Tera, Context};
use actix_web::{web, HttpResponse, Result};
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

// Structs สำหรับ template data
#[derive(Serialize)]
struct UserView {
    id: u64,
    username: String,
    email: String,
    is_active: bool,
    created_at: String,
    role: String,
    avatar_url: Option<String>,
}

#[derive(Serialize)]
struct PaginationView {
    total: u64,
    page: u64,
    per_page: u64,
    total_pages: u64,
    has_next: bool,
    has_prev: bool,
    next_page: Option<u64>,
    prev_page: Option<u64>,
}

#[derive(Serialize)]
struct StatsView {
    users: u64,
    posts: u64,
    comments: u64,
}

#[derive(Serialize)]
struct FeaturedItem {
    title: String,
    description: String,
    is_new: bool,
}

// Handler ที่ render template
async fn index_page(tera: web::Data<Tera>) -> Result<HttpResponse> {
    let mut context = Context::new();
    
    // เพิ่มข้อมูลพื้นฐาน
    context.insert("app_name", "My Actix App");
    context.insert("description", "เว็บแอปพลิเคชัน Rust + Actix-web");
    context.insert("current_date", &chrono::Utc::now().format("%d/%m/%Y").to_string());
    
    // เพิ่ม stats
    context.insert("stats", &StatsView {
        users: 150,
        posts: 420,
        comments: 1234,
    });
    
    // เพิ่ม featured items
    context.insert("featured_items", &vec![
        FeaturedItem {
            title: "บทความแนะนำ".to_string(),
            description: "เรียนรู้ Rust จากศูนย์".to_string(),
            is_new: true,
        },
        FeaturedItem {
            title: "Course ยอดนิยม".to_string(),
            description: "Actix-web สำหรับมือใหม่".to_string(),
            is_new: false,
        },
    ]);
    
    let rendered = tera.render("index.html", &context)
        .map_err(|e| {
            log::error!("Template render error: {}", e);
            actix_web::error::ErrorInternalServerError("Template error")
        })?;
    
    Ok(HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(rendered))
}

// Handler ที่แสดงรายการ users
async fn users_page(
    tera: web::Data<Tera>,
    query: web::Query<HashMap<String, String>>,
) -> Result<HttpResponse> {
    let page: u64 = query.get("page")
        .and_then(|p| p.parse().ok())
        .unwrap_or(1);
    
    // Simulate data
    let users: Vec<UserView> = (1..=10).map(|i| UserView {
        id: (page - 1) * 10 + i,
        username: format!("user_{}", (page - 1) * 10 + i),
        email: format!("user{}@example.com", (page - 1) * 10 + i),
        is_active: i % 5 != 0,
        created_at: "01/01/2024".to_string(),
        role: if i == 1 { "admin".to_string() } else { "user".to_string() },
        avatar_url: None,
    }).collect();
    
    let total = 100u64;
    let per_page = 10u64;
    let total_pages = (total + per_page - 1) / per_page;
    
    let mut context = Context::new();
    context.insert("users", &users);
    context.insert("pagination", &PaginationView {
        total,
        page,
        per_page,
        total_pages,
        has_next: page < total_pages,
        has_prev: page > 1,
        next_page: if page < total_pages { Some(page + 1) } else { None },
        prev_page: if page > 1 { Some(page - 1) } else { None },
    });
    
    let rendered = tera.render("users/list.html", &context)
        .map_err(|e| {
            log::error!("Template render error: {}", e);
            actix_web::error::ErrorInternalServerError("Template error")
        })?;
    
    Ok(HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(rendered))
}

// Helper function สำหรับสร้าง error response
fn render_error_page(
    tera: &Tera,
    status: u16,
    message: &str,
) -> HttpResponse {
    let mut context = Context::new();
    context.insert("status_code", &status);
    context.insert("error_message", message);
    
    match tera.render("errors/error.html", &context) {
        Ok(html) => HttpResponse::build(
            actix_web::http::StatusCode::from_u16(status).unwrap()
        )
        .content_type("text/html; charset=utf-8")
        .body(html),
        Err(_) => HttpResponse::InternalServerError()
            .body(format!("<h1>Error {}</h1><p>{}</p>", status, message))
    }
}
```

---

## HTML Forms

การจัดการ HTML forms ใน Actix-web

```rust
use actix_web::{web, HttpResponse, Result};
use serde::{Serialize, Deserialize};
use tera::{Tera, Context};

// Form data struct
#[derive(Debug, Deserialize)]
struct ContactForm {
    name: String,
    email: String,
    subject: String,
    message: String,
    phone: Option<String>,
}

#[derive(Debug, Deserialize)]
struct LoginForm {
    username: String,
    password: String,
    remember_me: Option<String>,  // checkbox
}

// Template สำหรับ contact form
fn create_contact_template() {
    let template = r#"{% extends "base.html" %}

{% block title %}ติดต่อเรา{% endblock %}

{% block content %}
<div class="card" style="max-width: 600px; margin: 0 auto;">
    <h1>ติดต่อเรา</h1>
    
    {% if errors %}
    <div class="alert alert-error">
        <ul>
        {% for error in errors %}
            <li>{{ error }}</li>
        {% endfor %}
        </ul>
    </div>
    {% endif %}
    
    <form method="POST" action="/contact" novalidate>
        <div class="form-group">
            <label for="name">ชื่อ-นามสกุล *</label>
            <input type="text" id="name" name="name" 
                   value="{{ form.name | default(value='') }}"
                   placeholder="กรอกชื่อ-นามสกุล" required>
            {% if field_errors.name %}
            <p class="error">{{ field_errors.name }}</p>
            {% endif %}
        </div>
        
        <div class="form-group">
            <label for="email">อีเมล *</label>
            <input type="email" id="email" name="email"
                   value="{{ form.email | default(value='') }}"
                   placeholder="example@email.com" required>
            {% if field_errors.email %}
            <p class="error">{{ field_errors.email }}</p>
            {% endif %}
        </div>
        
        <div class="form-group">
            <label for="phone">เบอร์โทรศัพท์</label>
            <input type="tel" id="phone" name="phone"
                   value="{{ form.phone | default(value='') }}"
                   placeholder="0x-xxxx-xxxx">
        </div>
        
        <div class="form-group">
            <label for="subject">หัวข้อ *</label>
            <select id="subject" name="subject" required>
                <option value="">-- เลือกหัวข้อ --</option>
                <option value="general" {% if form.subject == "general" %}selected{% endif %}>ทั่วไป</option>
                <option value="support" {% if form.subject == "support" %}selected{% endif %}>ขอความช่วยเหลือ</option>
                <option value="feedback" {% if form.subject == "feedback" %}selected{% endif %}>ข้อเสนอแนะ</option>
                <option value="business" {% if form.subject == "business" %}selected{% endif %}>ธุรกิจ</option>
            </select>
        </div>
        
        <div class="form-group">
            <label for="message">ข้อความ *</label>
            <textarea id="message" name="message" rows="6"
                      placeholder="กรอกข้อความของคุณ..." required>{{ form.message | default(value='') }}</textarea>
            {% if field_errors.message %}
            <p class="error">{{ field_errors.message }}</p>
            {% endif %}
        </div>
        
        <button type="submit" class="btn btn-primary">ส่งข้อความ</button>
    </form>
</div>
{% endblock %}"#;
    
    std::fs::create_dir_all("templates/forms").ok();
    std::fs::write("templates/forms/contact.html", template).ok();
}

// Handler แสดง form
async fn show_contact_form(tera: web::Data<Tera>) -> Result<HttpResponse> {
    let context = Context::new();
    
    let rendered = tera.render("forms/contact.html", &context)
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
    
    Ok(HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body(rendered))
}

// Handler รับ form submission
async fn submit_contact_form(
    tera: web::Data<Tera>,
    form: web::Form<ContactForm>,
) -> Result<HttpResponse> {
    let form_data = form.into_inner();
    
    // Validate
    let mut errors: Vec<String> = vec![];
    let mut field_errors: std::collections::HashMap<String, String> = std::collections::HashMap::new();
    
    if form_data.name.trim().is_empty() {
        field_errors.insert("name".to_string(), "กรุณากรอกชื่อ-นามสกุล".to_string());
    }
    
    if form_data.email.trim().is_empty() {
        field_errors.insert("email".to_string(), "กรุณากรอกอีเมล".to_string());
    } else if !form_data.email.contains('@') {
        field_errors.insert("email".to_string(), "รูปแบบอีเมลไม่ถูกต้อง".to_string());
    }
    
    if form_data.subject.trim().is_empty() {
        errors.push("กรุณาเลือกหัวข้อ".to_string());
    }
    
    if form_data.message.trim().is_empty() {
        field_errors.insert("message".to_string(), "กรุณากรอกข้อความ".to_string());
    } else if form_data.message.len() < 10 {
        field_errors.insert("message".to_string(), "ข้อความต้องมีอย่างน้อย 10 ตัวอักษร".to_string());
    }
    
    if !errors.is_empty() || !field_errors.is_empty() {
        let mut context = Context::new();
        context.insert("errors", &errors);
        context.insert("field_errors", &field_errors);
        context.insert("form", &serde_json::json!({
            "name": form_data.name,
            "email": form_data.email,
            "subject": form_data.subject,
            "message": form_data.message,
        }));
        
        let rendered = tera.render("forms/contact.html", &context)
            .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;
        
        return Ok(HttpResponse::BadRequest()
            .content_type("text/html; charset=utf-8")
            .body(rendered));
    }
    
    // Process form (ส่ง email, บันทึก database, ฯลฯ)
    log::info!("Contact form received from: {}", form_data.email);
    
    // Redirect พร้อม flash message
    Ok(HttpResponse::SeeOther()
        .append_header(("Location", "/contact?success=1"))
        .finish())
}
```

---

## File Download

การให้ users download ไฟล์

```rust
use actix_files::NamedFile;
use actix_web::{web, HttpRequest, HttpResponse, Result};
use actix_web::http::header::{ContentDisposition, DispositionType, DispositionParam};

// Download ไฟล์ที่มีอยู่แล้ว
async fn download_report(path: web::Path<String>) -> Result<NamedFile> {
    let report_name = path.into_inner();
    
    // Sanitize filename
    let safe_name: String = report_name.chars()
        .filter(|c| c.is_alphanumeric() || *c == '-' || *c == '_' || *c == '.')
        .collect();
    
    if safe_name.is_empty() || safe_name.contains("..") {
        return Err(actix_web::error::ErrorBadRequest("Invalid filename"));
    }
    
    let file_path = format!("storage/reports/{}", safe_name);
    
    let file = NamedFile::open(&file_path)
        .map_err(|_| actix_web::error::ErrorNotFound("Report not found"))?;
    
    Ok(file
        .set_content_disposition(ContentDisposition {
            disposition: DispositionType::Attachment,
            parameters: vec![DispositionParam::Filename(safe_name)],
        })
        .use_etag(true)
        .use_last_modified(true))
}

// สร้างและ download ไฟล์แบบ dynamic
async fn download_csv_export(
    query: web::Query<std::collections::HashMap<String, String>>,
) -> Result<HttpResponse> {
    // Simulate generating CSV data
    let mut csv = String::from("ID,Username,Email,Created At\n");
    
    for i in 1..=100 {
        csv.push_str(&format!(
            "{},user_{},user{}@example.com,2024-01-{:02}\n",
            i, i, i, (i % 28) + 1
        ));
    }
    
    let filename = format!("users_export_{}.csv", 
        chrono::Utc::now().format("%Y%m%d_%H%M%S"));
    
    Ok(HttpResponse::Ok()
        .content_type("text/csv; charset=utf-8")
        .append_header((
            "Content-Disposition",
            format!("attachment; filename=\"{}\"", filename)
        ))
        .append_header(("X-Content-Type-Options", "nosniff"))
        .body(csv))
}

// Download ไฟล์ใหญ่แบบ streaming
use actix_web::web::Bytes;
use futures_util::stream::Stream;
use std::pin::Pin;

async fn download_large_file(
    path: web::Path<String>,
) -> Result<HttpResponse> {
    let file_name = path.into_inner();
    let file_path = format!("storage/large_files/{}", file_name);
    
    let file = tokio::fs::File::open(&file_path)
        .await
        .map_err(|_| actix_web::error::ErrorNotFound("File not found"))?;
    
    let metadata = file.metadata().await
        .map_err(|_| actix_web::error::ErrorInternalServerError("Cannot read file metadata"))?;
    
    let file_size = metadata.len();
    
    // Stream the file
    let stream = tokio_util::io::ReaderStream::new(file);
    
    Ok(HttpResponse::Ok()
        .content_type("application/octet-stream")
        .append_header(("Content-Length", file_size.to_string()))
        .append_header((
            "Content-Disposition",
            format!("attachment; filename=\"{}\"", file_name)
        ))
        .streaming(stream))
}
```

---

## Content-Type Handling

การจัดการ MIME types ต่างๆ

```rust
use actix_web::{web, HttpResponse, Result};
use mime::Mime;
use std::str::FromStr;

// Handler ที่ return content type ต่างๆ
async fn serve_json() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("application/json")
        .body(r#"{"message": "Hello JSON"}"#)
}

async fn serve_xml() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("application/xml")
        .body(r#"<?xml version="1.0"?><message>Hello XML</message>"#)
}

async fn serve_html() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html; charset=utf-8")
        .body("<html><body><h1>Hello HTML</h1></body></html>")
}

async fn serve_plain_text() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/plain; charset=utf-8")
        .body("Hello Plain Text")
}

async fn serve_binary() -> HttpResponse {
    // ตัวอย่าง PNG image data (1x1 pixel transparent PNG)
    let png_data: Vec<u8> = vec![
        0x89, 0x50, 0x4E, 0x47, 0x0D, 0x0A, 0x1A, 0x0A,
        // ... minimal PNG data
    ];
    
    HttpResponse::Ok()
        .content_type("image/png")
        .body(png_data)
}

// Content negotiation ตาม Accept header
async fn negotiate_content(req: actix_web::HttpRequest) -> HttpResponse {
    let accept = req.headers()
        .get("Accept")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("*/*");
    
    if accept.contains("application/json") {
        return HttpResponse::Ok()
            .content_type("application/json")
            .json(serde_json::json!({"message": "Hello", "format": "JSON"}));
    }
    
    if accept.contains("text/html") {
        return HttpResponse::Ok()
            .content_type("text/html; charset=utf-8")
            .body("<html><body><p>Hello in HTML</p></body></html>");
    }
    
    if accept.contains("application/xml") {
        return HttpResponse::Ok()
            .content_type("application/xml")
            .body("<response><message>Hello in XML</message></response>");
    }
    
    // Default: JSON
    HttpResponse::Ok()
        .content_type("application/json")
        .json(serde_json::json!({"message": "Hello"}))
}

// Guess content type จาก file extension
fn get_content_type(filename: &str) -> &'static str {
    let ext = filename.rsplit('.').next().unwrap_or("").to_lowercase();
    match ext.as_str() {
        "html" | "htm" => "text/html; charset=utf-8",
        "css" => "text/css",
        "js" => "application/javascript",
        "json" => "application/json",
        "xml" => "application/xml",
        "png" => "image/png",
        "jpg" | "jpeg" => "image/jpeg",
        "gif" => "image/gif",
        "svg" => "image/svg+xml",
        "ico" => "image/x-icon",
        "webp" => "image/webp",
        "mp4" => "video/mp4",
        "mp3" => "audio/mpeg",
        "pdf" => "application/pdf",
        "zip" => "application/zip",
        "gz" => "application/gzip",
        "csv" => "text/csv",
        "txt" => "text/plain; charset=utf-8",
        "woff" => "font/woff",
        "woff2" => "font/woff2",
        "ttf" => "font/ttf",
        _ => "application/octet-stream",
    }
}
```

---

## ตัวอย่างสมบูรณ์: Full Web App

```rust
use actix_web::{App, HttpServer, web, middleware};
use actix_files as fs;
use tera::Tera;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(
        env_logger::Env::default().default_filter_or("info")
    );
    
    // สร้าง demo files
    create_demo_files();
    create_base_template();
    create_index_template();
    create_users_template();
    create_contact_template();
    
    let tera = web::Data::new(create_tera());
    
    HttpServer::new(move || {
        App::new()
            .app_data(tera.clone())
            .wrap(middleware::Logger::default())
            // Static files
            .service(
                fs::Files::new("/assets", "static/assets")
                    .use_etag(true)
                    .use_last_modified(true)
            )
            .service(
                fs::Files::new("/uploads", "storage/uploads")
            )
            // Pages
            .route("/", web::get().to(index_page))
            .route("/users", web::get().to(users_page))
            .route("/contact", web::get().to(show_contact_form))
            .route("/contact", web::post().to(submit_contact_form))
            // Downloads
            .route("/download/{filename}", web::get().to(download_report))
            .route("/export/users", web::get().to(download_csv_export))
            // API
            .service(
                web::scope("/api/v1")
                    .route("/users", web::get().to(|| async {
                        HttpResponse::Ok().json(serde_json::json!({"users": []}))
                    }))
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

1. **actix-files** - การ serve static files
2. **Static directory** - การ serve ทั้ง directory
3. **Single file** - การ serve ไฟล์เดี่ยวและ NamedFile
4. **Tera templates** - Server-side rendering ด้วย template engine
5. **Template context** - การส่งข้อมูลไปยัง templates
6. **HTML forms** - การรับและ validate form data
7. **File download** - การให้ download ไฟล์
8. **Content-Type** - การจัดการ MIME types

---

## การนำทาง

- [← Part 026: Error Handling in Actix-web](../part_026/README.md)
- [→ Part 028: CORS and Security Headers](../part_028/README.md)
- [กลับหน้าหลัก](../../README.md)
