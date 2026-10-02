# Part 055: Email Service 📧

## 🎯 เป้าหมายของ Part นี้

- lettre crate สำหรับส่ง email
- SMTP configuration
- Email templates ด้วย Tera
- HTML และ text emails
- Attachments
- Email verification flow
- Password reset email
- Email queuing
- สร้าง Complete Email Service

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
lettre = { version = "0.11", features = ["tokio1", "tokio1-native-tls", "builder", "smtp-transport", "hostname"] }
tera = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
base64 = "0.21"
sha2 = "0.10"
hmac = "0.12"
hex = "0.4"
anyhow = "1"
thiserror = "1"
dotenv = "0.15"
```

---

## 2. SMTP Configuration

```rust
// src/email/config.rs
use lettre::{
    transport::smtp::authentication::Credentials,
    AsyncSmtpTransport, Tokio1Executor,
};

#[derive(Debug, Clone)]
pub struct EmailConfig {
    pub smtp_host: String,
    pub smtp_port: u16,
    pub smtp_username: String,
    pub smtp_password: String,
    pub from_email: String,
    pub from_name: String,
    pub use_tls: bool,
}

impl EmailConfig {
    pub fn from_env() -> Self {
        Self {
            smtp_host: std::env::var("SMTP_HOST").unwrap_or("smtp.gmail.com".to_string()),
            smtp_port: std::env::var("SMTP_PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(587),
            smtp_username: std::env::var("SMTP_USERNAME").expect("SMTP_USERNAME required"),
            smtp_password: std::env::var("SMTP_PASSWORD").expect("SMTP_PASSWORD required"),
            from_email: std::env::var("FROM_EMAIL").expect("FROM_EMAIL required"),
            from_name: std::env::var("FROM_NAME").unwrap_or("My App".to_string()),
            use_tls: std::env::var("SMTP_USE_TLS")
                .map(|v| v == "true")
                .unwrap_or(true),
        }
    }

    pub fn build_transport(&self) -> anyhow::Result<AsyncSmtpTransport<Tokio1Executor>> {
        let creds = Credentials::new(
            self.smtp_username.clone(),
            self.smtp_password.clone(),
        );

        let transport = if self.use_tls {
            AsyncSmtpTransport::<Tokio1Executor>::starttls_relay(&self.smtp_host)?
                .credentials(creds)
                .port(self.smtp_port)
                .build()
        } else {
            AsyncSmtpTransport::<Tokio1Executor>::builder_dangerous(&self.smtp_host)
                .credentials(creds)
                .port(self.smtp_port)
                .build()
        };

        Ok(transport)
    }
}
```

---

## 3. Email Templates ด้วย Tera

```rust
// src/email/templates.rs
use tera::{Tera, Context};
use std::collections::HashMap;

pub struct EmailTemplateEngine {
    tera: Tera,
}

impl EmailTemplateEngine {
    pub fn new(template_dir: &str) -> anyhow::Result<Self> {
        let tera = Tera::new(&format!("{}/**/*.html", template_dir))?;
        Ok(Self { tera })
    }

    pub fn render(
        &self,
        template_name: &str,
        variables: HashMap<String, serde_json::Value>,
    ) -> anyhow::Result<String> {
        let mut context = Context::new();

        for (key, value) in variables {
            match value {
                serde_json::Value::String(s) => context.insert(&key, &s),
                serde_json::Value::Number(n) => {
                    if let Some(i) = n.as_i64() {
                        context.insert(&key, &i);
                    } else if let Some(f) = n.as_f64() {
                        context.insert(&key, &f);
                    }
                }
                serde_json::Value::Bool(b) => context.insert(&key, &b),
                v => context.insert(&key, &v.to_string()),
            }
        }

        Ok(self.tera.render(template_name, &context)?)
    }
}
```

```html
<!-- templates/email/verification.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial, sans-serif; background: #f4f4f4; }
        .container { max-width: 600px; margin: 40px auto; background: white; border-radius: 8px; padding: 40px; }
        .button {
            display: inline-block;
            padding: 12px 24px;
            background: #007bff;
            color: white;
            text-decoration: none;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>ยืนยัน Email ของคุณ</h1>
        <p>สวัสดีคุณ {{ username }},</p>
        <p>กรุณาคลิกปุ่มด้านล่างเพื่อยืนยัน email ของคุณ:</p>
        <a href="{{ verification_url }}" class="button">ยืนยัน Email</a>
        <p>ลิงก์นี้จะหมดอายุใน {{ expires_hours }} ชั่วโมง</p>
        <p>หากคุณไม่ได้สมัครสมาชิก กรุณาเพิกเฉยต่ออีเมลนี้</p>
    </div>
</body>
</html>
```

```html
<!-- templates/email/password_reset.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial, sans-serif; background: #f4f4f4; }
        .container { max-width: 600px; margin: 40px auto; background: white; border-radius: 8px; padding: 40px; }
        .button {
            display: inline-block;
            padding: 12px 24px;
            background: #dc3545;
            color: white;
            text-decoration: none;
            border-radius: 5px;
        }
        .warning { color: #856404; background: #fff3cd; padding: 10px; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>รีเซ็ตรหัสผ่าน</h1>
        <p>สวัสดีคุณ {{ username }},</p>
        <p>เราได้รับคำขอรีเซ็ตรหัสผ่านของคุณ</p>
        <a href="{{ reset_url }}" class="button">รีเซ็ตรหัสผ่าน</a>
        <p>ลิงก์นี้จะหมดอายุใน {{ expires_minutes }} นาที</p>
        <div class="warning">
            หากคุณไม่ได้ขอรีเซ็ตรหัสผ่าน กรุณาเปลี่ยนรหัสผ่านทันที
        </div>
    </div>
</body>
</html>
```

---

## 4. Email Builder

```rust
// src/email/builder.rs
use lettre::{
    message::{header::ContentType, Mailbox, Message, MultiPart, SinglePart},
    AsyncTransport, AsyncSmtpTransport, Tokio1Executor,
};
use std::collections::HashMap;

pub struct EmailBuilder {
    to: Vec<String>,
    cc: Vec<String>,
    bcc: Vec<String>,
    subject: String,
    html_body: Option<String>,
    text_body: Option<String>,
    attachments: Vec<Attachment>,
    reply_to: Option<String>,
}

pub struct Attachment {
    pub filename: String,
    pub content_type: String,
    pub data: Vec<u8>,
}

impl EmailBuilder {
    pub fn new() -> Self {
        Self {
            to: Vec::new(),
            cc: Vec::new(),
            bcc: Vec::new(),
            subject: String::new(),
            html_body: None,
            text_body: None,
            attachments: Vec::new(),
            reply_to: None,
        }
    }

    pub fn to(mut self, email: &str) -> Self {
        self.to.push(email.to_string());
        self
    }

    pub fn cc(mut self, email: &str) -> Self {
        self.cc.push(email.to_string());
        self
    }

    pub fn bcc(mut self, email: &str) -> Self {
        self.bcc.push(email.to_string());
        self
    }

    pub fn subject(mut self, subject: &str) -> Self {
        self.subject = subject.to_string();
        self
    }

    pub fn html_body(mut self, html: &str) -> Self {
        self.html_body = Some(html.to_string());
        self
    }

    pub fn text_body(mut self, text: &str) -> Self {
        self.text_body = Some(text.to_string());
        self
    }

    pub fn attachment(mut self, filename: &str, content_type: &str, data: Vec<u8>) -> Self {
        self.attachments.push(Attachment {
            filename: filename.to_string(),
            content_type: content_type.to_string(),
            data,
        });
        self
    }

    pub fn build(
        self,
        from_email: &str,
        from_name: &str,
    ) -> anyhow::Result<Message> {
        let from: Mailbox = format!("{} <{}>", from_name, from_email).parse()?;

        let mut builder = Message::builder()
            .from(from)
            .subject(&self.subject);

        for to in &self.to {
            builder = builder.to(to.parse()?);
        }

        for cc in &self.cc {
            builder = builder.cc(cc.parse()?);
        }

        // Build body
        let body = match (self.html_body, self.text_body) {
            (Some(html), Some(text)) => {
                MultiPart::alternative()
                    .singlepart(
                        SinglePart::builder()
                            .header(ContentType::TEXT_PLAIN)
                            .body(text),
                    )
                    .singlepart(
                        SinglePart::builder()
                            .header(ContentType::TEXT_HTML)
                            .body(html),
                    )
            }
            (Some(html), None) => {
                MultiPart::alternative().singlepart(
                    SinglePart::builder()
                        .header(ContentType::TEXT_HTML)
                        .body(html),
                )
            }
            (None, Some(text)) => {
                MultiPart::alternative().singlepart(
                    SinglePart::builder()
                        .header(ContentType::TEXT_PLAIN)
                        .body(text),
                )
            }
            (None, None) => {
                MultiPart::alternative().singlepart(
                    SinglePart::builder()
                        .header(ContentType::TEXT_PLAIN)
                        .body(String::new()),
                )
            }
        };

        // Add attachments
        let final_body = if self.attachments.is_empty() {
            body
        } else {
            let mut mixed = MultiPart::mixed().multipart(body);

            for att in self.attachments {
                let ct: ContentType = att.content_type.parse()?;
                mixed = mixed.singlepart(
                    lettre::message::Attachment::new(att.filename)
                        .body(att.data, ct),
                );
            }

            mixed
        };

        Ok(builder.multipart(final_body)?)
    }
}
```

---

## 5. Email Service

```rust
// src/email/service.rs
use std::sync::Arc;
use lettre::{AsyncSmtpTransport, AsyncTransport, Tokio1Executor};
use super::{config::EmailConfig, builder::EmailBuilder, templates::EmailTemplateEngine};

pub struct EmailService {
    transport: Arc<AsyncSmtpTransport<Tokio1Executor>>,
    config: EmailConfig,
    templates: Arc<EmailTemplateEngine>,
}

impl EmailService {
    pub async fn new(config: EmailConfig) -> anyhow::Result<Self> {
        let transport = config.build_transport()?;
        let templates = EmailTemplateEngine::new("./templates/email")?;

        Ok(Self {
            transport: Arc::new(transport),
            config,
            templates: Arc::new(templates),
        })
    }

    pub async fn send_raw(
        &self,
        builder: EmailBuilder,
    ) -> anyhow::Result<()> {
        let message = builder.build(&self.config.from_email, &self.config.from_name)?;
        self.transport.send(message).await?;
        Ok(())
    }

    // ส่ง email verification
    pub async fn send_verification_email(
        &self,
        to_email: &str,
        username: &str,
        verification_token: &str,
        base_url: &str,
    ) -> anyhow::Result<()> {
        let verification_url = format!(
            "{}/auth/verify?token={}",
            base_url, verification_token
        );

        let mut vars = std::collections::HashMap::new();
        vars.insert("username".to_string(), serde_json::json!(username));
        vars.insert("verification_url".to_string(), serde_json::json!(verification_url));
        vars.insert("expires_hours".to_string(), serde_json::json!(24));

        let html = self.templates.render("verification.html", vars.clone())?;

        let builder = EmailBuilder::new()
            .to(to_email)
            .subject("ยืนยัน Email ของคุณ")
            .html_body(&html)
            .text_body(&format!(
                "กรุณาคลิกลิงก์เพื่อยืนยัน email: {}",
                verification_url
            ));

        self.send_raw(builder).await
    }

    // ส่ง password reset
    pub async fn send_password_reset(
        &self,
        to_email: &str,
        username: &str,
        reset_token: &str,
        base_url: &str,
    ) -> anyhow::Result<()> {
        let reset_url = format!(
            "{}/auth/reset-password?token={}",
            base_url, reset_token
        );

        let mut vars = std::collections::HashMap::new();
        vars.insert("username".to_string(), serde_json::json!(username));
        vars.insert("reset_url".to_string(), serde_json::json!(reset_url));
        vars.insert("expires_minutes".to_string(), serde_json::json!(30));

        let html = self.templates.render("password_reset.html", vars)?;

        let builder = EmailBuilder::new()
            .to(to_email)
            .subject("รีเซ็ตรหัสผ่าน")
            .html_body(&html);

        self.send_raw(builder).await
    }

    // ส่ง email พร้อม attachment
    pub async fn send_with_attachment(
        &self,
        to_email: &str,
        subject: &str,
        body_html: &str,
        attachment_name: &str,
        attachment_data: Vec<u8>,
        attachment_type: &str,
    ) -> anyhow::Result<()> {
        let builder = EmailBuilder::new()
            .to(to_email)
            .subject(subject)
            .html_body(body_html)
            .attachment(attachment_name, attachment_type, attachment_data);

        self.send_raw(builder).await
    }
}
```

---

## 6. Email Verification Flow

```rust
// src/auth/email_verify.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use sha2::{Sha256, Digest};
use hex;
use chrono::{Utc, Duration};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EmailVerificationToken {
    pub token: String,
    pub user_id: String,
    pub email: String,
    pub expires_at: chrono::DateTime<chrono::Utc>,
    pub used: bool,
}

impl EmailVerificationToken {
    pub fn new(user_id: &str, email: &str) -> Self {
        let raw = format!("{}-{}-{}", user_id, email, Uuid::new_v4());
        let token = hex::encode(Sha256::digest(raw.as_bytes()));

        Self {
            token,
            user_id: user_id.to_string(),
            email: email.to_string(),
            expires_at: Utc::now() + Duration::hours(24),
            used: false,
        }
    }

    pub fn is_valid(&self) -> bool {
        !self.used && self.expires_at > Utc::now()
    }
}

#[derive(Deserialize)]
pub struct VerifyEmailQuery {
    pub token: String,
}

pub async fn verify_email(
    query: web::Query<VerifyEmailQuery>,
    email_service: web::Data<EmailService>,
    // db: web::Data<Db>,
) -> HttpResponse {
    // ค้นหา token ใน database
    // let token_record = db.find_verification_token(&query.token).await...;

    // ตรวจสอบความถูกต้อง
    // if !token_record.is_valid() { return error; }

    // อัพเดต user verified status
    // db.verify_user_email(&token_record.user_id).await...;

    // Mark token as used
    // db.mark_token_used(&query.token).await...;

    HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "message": "Email verified successfully"
    }))
}

use super::service::EmailService;

pub async fn resend_verification(
    email_service: web::Data<EmailService>,
    body: web::Json<ResendVerificationRequest>,
) -> HttpResponse {
    // สร้าง token ใหม่
    let token = EmailVerificationToken::new("user_id", &body.email);

    // บันทึก token ใน database
    // db.save_verification_token(&token).await...;

    // ส่ง email
    match email_service
        .send_verification_email(
            &body.email,
            &body.username,
            &token.token,
            "https://myapp.com",
        )
        .await
    {
        Ok(_) => HttpResponse::Ok().json(serde_json::json!({
            "success": true,
            "message": "Verification email sent"
        })),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({
            "error": format!("Failed to send email: {}", e)
        })),
    }
}

#[derive(Deserialize)]
pub struct ResendVerificationRequest {
    pub email: String,
    pub username: String,
}
```

---

## 7. Email Queue

```rust
// src/email/queue.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::{mpsc, RwLock};
use uuid::Uuid;
use super::service::EmailService;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QueuedEmail {
    pub id: String,
    pub to: String,
    pub subject: String,
    pub html_body: String,
    pub text_body: Option<String>,
    pub priority: u8, // 0=low, 1=normal, 2=high
    pub queued_at: chrono::DateTime<chrono::Utc>,
    pub status: EmailStatus,
    pub attempts: u32,
    pub error: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum EmailStatus {
    Queued,
    Sending,
    Sent,
    Failed,
}

pub struct EmailQueue {
    queue: Arc<RwLock<Vec<QueuedEmail>>>,
    sender: mpsc::Sender<String>, // email id
    email_service: Arc<EmailService>,
}

impl EmailQueue {
    pub fn new(email_service: Arc<EmailService>) -> Self {
        let queue: Arc<RwLock<Vec<QueuedEmail>>> = Arc::new(RwLock::new(Vec::new()));
        let (tx, mut rx) = mpsc::channel::<String>(500);

        let queue_clone = queue.clone();
        let service = email_service.clone();

        tokio::spawn(async move {
            while let Some(email_id) = rx.recv().await {
                // หา email จาก queue
                let email = {
                    let q = queue_clone.read().await;
                    q.iter().find(|e| e.id == email_id).cloned()
                };

                if let Some(email) = email {
                    // อัพเดต status
                    {
                        let mut q = queue_clone.write().await;
                        if let Some(e) = q.iter_mut().find(|e| e.id == email_id) {
                            e.status = EmailStatus::Sending;
                        }
                    }

                    // ส่ง email
                    use super::builder::EmailBuilder;
                    let builder = EmailBuilder::new()
                        .to(&email.to)
                        .subject(&email.subject)
                        .html_body(&email.html_body);

                    let result = service.send_raw(builder).await;

                    // อัพเดต status ตาม result
                    let mut q = queue_clone.write().await;
                    if let Some(e) = q.iter_mut().find(|e| e.id == email_id) {
                        match result {
                            Ok(_) => {
                                e.status = EmailStatus::Sent;
                                println!("[EmailQueue] Sent: {}", email_id);
                            }
                            Err(err) => {
                                e.attempts += 1;
                                e.error = Some(err.to_string());
                                e.status = if e.attempts >= 3 {
                                    EmailStatus::Failed
                                } else {
                                    EmailStatus::Queued
                                };
                            }
                        }
                    }
                }
            }
        });

        Self {
            queue,
            sender: tx,
            email_service,
        }
    }

    pub async fn enqueue(&self, email: QueuedEmail) -> String {
        let id = email.id.clone();
        self.queue.write().await.push(email);
        let _ = self.sender.send(id.clone()).await;
        id
    }

    pub async fn get_stats(&self) -> serde_json::Value {
        let q = self.queue.read().await;
        serde_json::json!({
            "total": q.len(),
            "queued": q.iter().filter(|e| e.status == EmailStatus::Queued).count(),
            "sent": q.iter().filter(|e| e.status == EmailStatus::Sent).count(),
            "failed": q.iter().filter(|e| e.status == EmailStatus::Failed).count(),
        })
    }
}

// API handler
pub async fn queue_email(
    email_queue: web::Data<EmailQueue>,
    body: web::Json<QueueEmailRequest>,
) -> HttpResponse {
    let email = QueuedEmail {
        id: Uuid::new_v4().to_string(),
        to: body.to.clone(),
        subject: body.subject.clone(),
        html_body: body.html_body.clone(),
        text_body: body.text_body.clone(),
        priority: body.priority.unwrap_or(1),
        queued_at: chrono::Utc::now(),
        status: EmailStatus::Queued,
        attempts: 0,
        error: None,
    };

    let email_id = email_queue.enqueue(email).await;

    HttpResponse::Accepted().json(serde_json::json!({
        "email_id": email_id,
        "status": "queued"
    }))
}

#[derive(Deserialize)]
pub struct QueueEmailRequest {
    pub to: String,
    pub subject: String,
    pub html_body: String,
    pub text_body: Option<String>,
    pub priority: Option<u8>,
}

use serde::Deserialize;
```

---

## 8. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer};
use std::sync::Arc;

mod email {
    pub mod config;
    pub mod service;
    pub mod builder;
    pub mod templates;
    pub mod queue;
}

mod auth {
    pub mod email_verify;
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();

    let email_config = email::config::EmailConfig::from_env();
    let email_service = Arc::new(
        email::service::EmailService::new(email_config)
            .await
            .expect("Failed to create email service"),
    );

    let email_queue = web::Data::new(
        email::queue::EmailQueue::new(email_service.clone())
    );

    let email_service_data = web::Data::from(email_service);

    HttpServer::new(move || {
        App::new()
            .app_data(email_service_data.clone())
            .app_data(email_queue.clone())
            .route("/auth/verify", web::get().to(auth::email_verify::verify_email))
            .route("/auth/resend-verification", web::post().to(auth::email_verify::resend_verification))
            .route("/api/emails/queue", web::post().to(email::queue::queue_email))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

✅ lettre crate setup  
✅ SMTP configuration  
✅ Email templates ด้วย Tera  
✅ HTML + text emails  
✅ Email attachments  
✅ Email verification flow  
✅ Password reset email  
✅ Email queue system  

### Exercise

1. เพิ่ม email tracking (open, click tracking)
2. สร้าง email template editor
3. Implement bulk email sending
4. เพิ่ม bounce/complaint handling

---

*[← Part 054: Background Jobs with Tokio](../part_054/README.md) | [Part 056: Payment Gateway Integration →](../part_056/README.md)*
