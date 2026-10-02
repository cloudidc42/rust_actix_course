# Part 086: Project: Notification Service 🔔

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Multi-channel Notification Service
- ส่ง notifications ผ่าน Email, SMS, Push, WebSocket
- ออกแบบ Notification Templates
- จัดการ Notification Preferences
- ทำ Delivery Tracking
- สร้าง Retry Mechanism
- ทำ Batch Notifications
- จัดการ Unsubscribe
- สร้าง Complete Microservice

---

## 1. โครงสร้างโปรเจกต์

```
notification_service/
├── Cargo.toml
├── .env
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── notification.rs
    │   ├── template.rs
    │   └── preference.rs
    ├── channels/
    │   ├── mod.rs
    │   ├── email.rs
    │   ├── sms.rs
    │   ├── push.rs
    │   └── websocket.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── notifications.rs
    │   ├── templates.rs
    │   └── preferences.rs
    └── workers/
        ├── mod.rs
        ├── delivery.rs
        └── retry.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "notification_service"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
actix-web-actors = "4"
actix = "0.13"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
thiserror = "1"
reqwest = { version = "0.12", features = ["json"] }
handlebars = "6"
lettre = { version = "0.11", features = ["tokio1-native-tls", "builder"] }
redis = { version = "0.26", features = ["tokio-comp", "connection-manager"] }
tokio-cron-scheduler = "0.10"
validator = { version = "0.18", features = ["derive"] }
tera = "1"
```

---

## 3. Models

### 3.1 Notification Model (`src/models/notification.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "notification_channel", rename_all = "lowercase")]
pub enum NotificationChannel {
    Email,
    Sms,
    Push,
    WebSocket,
    Webhook,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "notification_status", rename_all = "lowercase")]
pub enum NotificationStatus {
    Pending,
    Queued,
    Sending,
    Sent,
    Delivered,
    Failed,
    Bounced,
    Unsubscribed,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "notification_priority", rename_all = "lowercase")]
pub enum NotificationPriority {
    Low,
    Normal,
    High,
    Critical,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Notification {
    pub id: Uuid,
    pub user_id: Option<Uuid>,
    pub channel: NotificationChannel,
    pub recipient: String,
    pub subject: Option<String>,
    pub content: String,
    pub html_content: Option<String>,
    pub template_id: Option<Uuid>,
    pub template_data: Option<serde_json::Value>,
    pub status: NotificationStatus,
    pub priority: NotificationPriority,
    pub scheduled_at: Option<DateTime<Utc>>,
    pub sent_at: Option<DateTime<Utc>>,
    pub delivered_at: Option<DateTime<Utc>>,
    pub retry_count: i32,
    pub max_retries: i32,
    pub next_retry_at: Option<DateTime<Utc>>,
    pub error_message: Option<String>,
    pub metadata: Option<serde_json::Value>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct SendNotificationRequest {
    pub user_id: Option<Uuid>,
    pub channel: NotificationChannel,
    pub recipient: String,
    pub subject: Option<String>,
    pub content: Option<String>,
    pub template_id: Option<Uuid>,
    pub template_data: Option<serde_json::Value>,
    pub priority: Option<NotificationPriority>,
    pub scheduled_at: Option<DateTime<Utc>>,
    pub metadata: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize)]
pub struct BatchNotificationRequest {
    pub notifications: Vec<SendNotificationRequest>,
    pub group_id: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct NotificationResult {
    pub notification_id: Uuid,
    pub status: NotificationStatus,
    pub message: String,
}
```

### 3.2 Template Model (`src/models/template.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct NotificationTemplate {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub channel: super::notification::NotificationChannel,
    pub subject_template: Option<String>,
    pub body_template: String,
    pub html_template: Option<String>,
    pub variables: Vec<String>,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateTemplateRequest {
    pub name: String,
    pub slug: String,
    pub channel: super::notification::NotificationChannel,
    pub subject_template: Option<String>,
    pub body_template: String,
    pub html_template: Option<String>,
    pub variables: Option<Vec<String>>,
}
```

### 3.3 Preference Model (`src/models/preference.rs`)

```rust
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct NotificationPreference {
    pub user_id: Uuid,
    pub notification_type: String,
    pub email_enabled: bool,
    pub sms_enabled: bool,
    pub push_enabled: bool,
    pub websocket_enabled: bool,
    pub email_address: Option<String>,
    pub phone_number: Option<String>,
    pub push_token: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdatePreferenceRequest {
    pub notification_type: String,
    pub email_enabled: Option<bool>,
    pub sms_enabled: Option<bool>,
    pub push_enabled: Option<bool>,
    pub websocket_enabled: Option<bool>,
    pub email_address: Option<String>,
    pub phone_number: Option<String>,
    pub push_token: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UnsubscribeRequest {
    pub user_id: Uuid,
    pub notification_type: Option<String>,
    pub channel: Option<String>,
}
```

---

## 4. Notification Channels

### 4.1 Email Channel (`src/channels/email.rs`)

```rust
use lettre::{
    message::{header::ContentType, Mailbox, Message, MultiPart, SinglePart},
    transport::smtp::{authentication::Credentials, response::Response},
    AsyncSmtpTransport, AsyncTransport, Tokio1Executor,
};
use std::str::FromStr;

use crate::errors::AppError;

pub struct EmailChannel {
    mailer: AsyncSmtpTransport<Tokio1Executor>,
    from_email: String,
    from_name: String,
}

impl EmailChannel {
    pub fn new(
        smtp_host: &str,
        smtp_port: u16,
        username: &str,
        password: &str,
        from_email: &str,
        from_name: &str,
    ) -> Result<Self, AppError> {
        let creds = Credentials::new(username.to_string(), password.to_string());

        let mailer = AsyncSmtpTransport::<Tokio1Executor>::relay(smtp_host)
            .map_err(|e| AppError::InternalError(format!("SMTP relay error: {}", e)))?
            .port(smtp_port)
            .credentials(creds)
            .build();

        Ok(EmailChannel {
            mailer,
            from_email: from_email.to_string(),
            from_name: from_name.to_string(),
        })
    }

    pub async fn send(
        &self,
        to: &str,
        subject: &str,
        text_body: &str,
        html_body: Option<&str>,
    ) -> Result<String, AppError> {
        let from = format!("{} <{}>", self.from_name, self.from_email)
            .parse::<Mailbox>()
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let to_mailbox = to.parse::<Mailbox>()
            .map_err(|e| AppError::BadRequest(format!("Invalid email: {}", e)))?;

        let message = if let Some(html) = html_body {
            Message::builder()
                .from(from)
                .to(to_mailbox)
                .subject(subject)
                .multipart(
                    MultiPart::alternative()
                        .singlepart(
                            SinglePart::builder()
                                .header(ContentType::TEXT_PLAIN)
                                .body(text_body.to_string()),
                        )
                        .singlepart(
                            SinglePart::builder()
                                .header(ContentType::TEXT_HTML)
                                .body(html.to_string()),
                        ),
                )
                .map_err(|e| AppError::InternalError(e.to_string()))?
        } else {
            Message::builder()
                .from(from)
                .to(to_mailbox)
                .subject(subject)
                .header(ContentType::TEXT_PLAIN)
                .body(text_body.to_string())
                .map_err(|e| AppError::InternalError(e.to_string()))?
        };

        let response = self
            .mailer
            .send(message)
            .await
            .map_err(|e| AppError::InternalError(format!("Email send error: {}", e)))?;

        log::info!("Email sent to {}: {:?}", to, response.code());

        Ok(format!("Email sent successfully to {}", to))
    }
}
```

### 4.2 SMS Channel - Mock Twilio (`src/channels/sms.rs`)

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};

use crate::errors::AppError;

pub struct SmsChannel {
    account_sid: String,
    auth_token: String,
    from_number: String,
    client: Client,
    is_mock: bool,
}

impl SmsChannel {
    pub fn new(account_sid: String, auth_token: String, from_number: String) -> Self {
        SmsChannel {
            account_sid,
            auth_token,
            from_number,
            client: Client::new(),
            is_mock: account_sid.starts_with("MOCK"),
        }
    }

    pub async fn send(&self, to: &str, body: &str) -> Result<String, AppError> {
        if self.is_mock {
            log::info!("[MOCK SMS] To: {}, Body: {}", to, body);
            return Ok(format!("mock_sms_{}", uuid::Uuid::new_v4()));
        }

        let url = format!(
            "https://api.twilio.com/2010-04-01/Accounts/{}/Messages.json",
            self.account_sid
        );

        let params = [
            ("To", to),
            ("From", &self.from_number),
            ("Body", body),
        ];

        let response = self
            .client
            .post(&url)
            .basic_auth(&self.account_sid, Some(&self.auth_token))
            .form(&params)
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("SMS request failed: {}", e)))?;

        if !response.status().is_success() {
            let error = response.text().await.unwrap_or_default();
            return Err(AppError::InternalError(format!("SMS send failed: {}", error)));
        }

        let result: serde_json::Value = response.json().await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        let sid = result["sid"].as_str().unwrap_or("").to_string();
        log::info!("SMS sent to {}, SID: {}", to, sid);
        Ok(sid)
    }
}
```

### 4.3 Push Notification Channel (`src/channels/push.rs`)

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use serde_json::json;

use crate::errors::AppError;

#[derive(Debug, Serialize)]
pub struct PushPayload {
    pub title: String,
    pub body: String,
    pub data: Option<serde_json::Value>,
    pub badge: Option<u32>,
    pub sound: Option<String>,
}

pub struct PushChannel {
    fcm_server_key: String,
    apns_key: Option<String>,
    client: Client,
}

impl PushChannel {
    pub fn new(fcm_server_key: String) -> Self {
        PushChannel {
            fcm_server_key,
            apns_key: None,
            client: Client::new(),
        }
    }

    pub async fn send_fcm(
        &self,
        device_token: &str,
        payload: &PushPayload,
    ) -> Result<String, AppError> {
        if self.fcm_server_key.starts_with("MOCK") {
            log::info!("[MOCK PUSH] Token: {}, Title: {}", device_token, payload.title);
            return Ok(format!("mock_push_{}", uuid::Uuid::new_v4()));
        }

        let body = json!({
            "to": device_token,
            "notification": {
                "title": payload.title,
                "body": payload.body,
                "badge": payload.badge,
                "sound": payload.sound.as_deref().unwrap_or("default"),
            },
            "data": payload.data,
        });

        let response = self
            .client
            .post("https://fcm.googleapis.com/fcm/send")
            .header("Authorization", format!("key={}", self.fcm_server_key))
            .header("Content-Type", "application/json")
            .json(&body)
            .send()
            .await
            .map_err(|e| AppError::InternalError(format!("FCM request failed: {}", e)))?;

        let result: serde_json::Value = response.json().await
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        if result["success"].as_i64().unwrap_or(0) == 0 {
            let error = result["results"][0]["error"].as_str().unwrap_or("Unknown");
            return Err(AppError::InternalError(format!("Push failed: {}", error)));
        }

        Ok(result["multicast_id"].to_string())
    }
}
```

---

## 5. Template Engine Service

```rust
// src/services/template_service.rs
use tera::{Context, Tera};
use uuid::Uuid;
use sqlx::PgPool;

use crate::errors::AppError;
use crate::models::template::NotificationTemplate;

pub struct TemplateService {
    tera: Tera,
}

impl TemplateService {
    pub fn new() -> Self {
        TemplateService {
            tera: Tera::default(),
        }
    }

    pub fn render_template(
        &self,
        template_str: &str,
        data: &serde_json::Value,
    ) -> Result<String, AppError> {
        let mut tera = Tera::default();
        tera.add_raw_template("notification", template_str)
            .map_err(|e| AppError::InternalError(format!("Template parse error: {}", e)))?;

        let mut context = Context::new();
        if let Some(obj) = data.as_object() {
            for (key, value) in obj {
                context.insert(key, value);
            }
        }

        tera.render("notification", &context)
            .map_err(|e| AppError::InternalError(format!("Template render error: {}", e)))
    }

    pub async fn get_and_render(
        pool: &PgPool,
        template_id: Uuid,
        data: &serde_json::Value,
    ) -> Result<RenderedNotification, AppError> {
        let template = sqlx::query_as!(
            NotificationTemplate,
            r#"
            SELECT id, name, slug, channel as "channel: _", subject_template,
                   body_template, html_template, variables, is_active,
                   created_at, updated_at
            FROM notification_templates
            WHERE id = $1 AND is_active = true
            "#,
            template_id
        )
        .fetch_optional(pool)
        .await?
        .ok_or_else(|| AppError::NotFound("Template not found".to_string()))?;

        let service = Self::new();

        let subject = if let Some(subject_tmpl) = &template.subject_template {
            Some(service.render_template(subject_tmpl, data)?)
        } else {
            None
        };

        let body = service.render_template(&template.body_template, data)?;

        let html = if let Some(html_tmpl) = &template.html_template {
            Some(service.render_template(html_tmpl, data)?)
        } else {
            None
        };

        Ok(RenderedNotification { subject, body, html })
    }
}

#[derive(Debug)]
pub struct RenderedNotification {
    pub subject: Option<String>,
    pub body: String,
    pub html: Option<String>,
}
```

---

## 6. Delivery Worker (`src/workers/delivery.rs`)

```rust
use sqlx::PgPool;
use std::sync::Arc;
use tokio::time::{interval, Duration};
use uuid::Uuid;

use crate::channels::email::EmailChannel;
use crate::channels::push::PushChannel;
use crate::channels::sms::SmsChannel;
use crate::errors::AppError;
use crate::models::notification::{NotificationChannel, NotificationStatus};

pub struct DeliveryWorker {
    pool: PgPool,
    email: Arc<EmailChannel>,
    sms: Arc<SmsChannel>,
    push: Arc<PushChannel>,
}

impl DeliveryWorker {
    pub fn new(
        pool: PgPool,
        email: EmailChannel,
        sms: SmsChannel,
        push: PushChannel,
    ) -> Self {
        DeliveryWorker {
            pool,
            email: Arc::new(email),
            sms: Arc::new(sms),
            push: Arc::new(push),
        }
    }

    pub async fn start(self) {
        let mut ticker = interval(Duration::from_secs(5));
        loop {
            ticker.tick().await;
            if let Err(e) = self.process_pending().await {
                log::error!("Delivery worker error: {}", e);
            }
        }
    }

    async fn process_pending(&self) -> Result<(), AppError> {
        // Fetch up to 50 pending notifications
        let notifications = sqlx::query!(
            r#"
            SELECT id, channel::text, recipient, subject, content, html_content,
                   template_id, template_data, retry_count, max_retries
            FROM notifications
            WHERE status IN ('pending', 'queued')
              AND (scheduled_at IS NULL OR scheduled_at <= NOW())
            ORDER BY priority DESC, created_at ASC
            LIMIT 50
            FOR UPDATE SKIP LOCKED
            "#
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| AppError::DatabaseError(e.to_string()))?;

        for notif in notifications {
            // Mark as sending
            sqlx::query!(
                "UPDATE notifications SET status = 'sending', updated_at = NOW() WHERE id = $1",
                notif.id
            )
            .execute(&self.pool)
            .await?;

            let result = match notif.channel.as_deref() {
                Some("email") => {
                    self.email.send(
                        &notif.recipient,
                        notif.subject.as_deref().unwrap_or("Notification"),
                        &notif.content,
                        notif.html_content.as_deref(),
                    ).await
                }
                Some("sms") => {
                    self.sms.send(&notif.recipient, &notif.content).await
                }
                Some("push") => {
                    let payload = crate::channels::push::PushPayload {
                        title: notif.subject.clone().unwrap_or("Notification".to_string()),
                        body: notif.content.clone(),
                        data: None,
                        badge: None,
                        sound: None,
                    };
                    self.push.send_fcm(&notif.recipient, &payload).await
                }
                _ => Err(AppError::BadRequest("Unsupported channel".to_string())),
            };

            match result {
                Ok(_) => {
                    sqlx::query!(
                        r#"
                        UPDATE notifications
                        SET status = 'sent', sent_at = NOW(), updated_at = NOW()
                        WHERE id = $1
                        "#,
                        notif.id
                    )
                    .execute(&self.pool)
                    .await?;
                }
                Err(e) => {
                    let retry_count = notif.retry_count + 1;
                    let new_status = if retry_count >= notif.max_retries {
                        "failed"
                    } else {
                        "pending"
                    };

                    // Exponential backoff
                    let next_retry = chrono::Utc::now()
                        + chrono::Duration::minutes(2i64.pow(retry_count as u32));

                    sqlx::query!(
                        r#"
                        UPDATE notifications
                        SET status = $1::notification_status,
                            retry_count = $2,
                            next_retry_at = $3,
                            error_message = $4,
                            updated_at = NOW()
                        WHERE id = $5
                        "#,
                        new_status,
                        retry_count,
                        if retry_count < notif.max_retries { Some(next_retry) } else { None },
                        e.to_string(),
                        notif.id
                    )
                    .execute(&self.pool)
                    .await?;

                    log::error!(
                        "Notification {} failed (attempt {}/{}): {}",
                        notif.id, retry_count, notif.max_retries, e
                    );
                }
            }
        }

        Ok(())
    }
}
```

---

## 7. Handlers

### Notifications Handler (`src/handlers/notifications.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::notification::{
    BatchNotificationRequest, NotificationPriority, NotificationStatus, SendNotificationRequest,
};
use crate::services::template_service::TemplateService;

pub async fn send_notification(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<SendNotificationRequest>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;

    // Check user preferences if user_id provided
    if let Some(user_id) = body.user_id {
        let pref = sqlx::query!(
            "SELECT * FROM notification_preferences WHERE user_id = $1",
            user_id
        )
        .fetch_optional(pool.get_ref())
        .await?;

        // Check if user has unsubscribed from this channel
        if let Some(pref) = &pref {
            let channel_enabled = match body.channel {
                crate::models::notification::NotificationChannel::Email => pref.email_enabled,
                crate::models::notification::NotificationChannel::Sms => pref.sms_enabled,
                crate::models::notification::NotificationChannel::Push => pref.push_enabled,
                _ => true,
            };

            if !channel_enabled {
                return Ok(HttpResponse::Ok().json(serde_json::json!({
                    "status": "unsubscribed",
                    "message": "User has unsubscribed from this notification channel",
                })));
            }
        }
    }

    // Render template if provided
    let (content, html_content, subject) = if let Some(template_id) = body.template_id {
        let data = body.template_data.clone().unwrap_or(serde_json::json!({}));
        let rendered = TemplateService::get_and_render(pool.get_ref(), template_id, &data).await?;
        (rendered.body, rendered.html, rendered.subject)
    } else {
        (
            body.content.clone().unwrap_or_default(),
            None,
            body.subject.clone(),
        )
    };

    let notif_id = Uuid::new_v4();
    let channel_str = format!("{:?}", body.channel).to_lowercase();
    let priority_str = body.priority.as_ref()
        .map(|p| format!("{:?}", p).to_lowercase())
        .unwrap_or_else(|| "normal".to_string());

    sqlx::query!(
        r#"
        INSERT INTO notifications (id, user_id, channel, recipient, subject, content, html_content,
                                  template_id, template_data, status, priority, scheduled_at, metadata)
        VALUES ($1, $2, $3::notification_channel, $4, $5, $6, $7, $8, $9,
                'pending'::notification_status, $10::notification_priority, $11, $12)
        "#,
        notif_id,
        body.user_id,
        channel_str,
        body.recipient,
        subject,
        content,
        html_content,
        body.template_id,
        body.template_data,
        priority_str,
        body.scheduled_at,
        body.metadata
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Created().json(serde_json::json!({
        "notification_id": notif_id,
        "status": "queued",
        "message": "Notification queued for delivery",
    })))
}

pub async fn send_batch(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<BatchNotificationRequest>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;

    if body.notifications.is_empty() || body.notifications.len() > 1000 {
        return Err(AppError::BadRequest("Batch size must be between 1 and 1000".to_string()));
    }

    let mut results = Vec::new();
    let mut tx = pool.begin().await?;

    for notif_req in &body.notifications {
        let notif_id = Uuid::new_v4();
        let channel_str = format!("{:?}", notif_req.channel).to_lowercase();

        let insert_result = sqlx::query!(
            r#"
            INSERT INTO notifications (id, user_id, channel, recipient, subject, content,
                                      template_id, template_data, status, metadata)
            VALUES ($1, $2, $3::notification_channel, $4, $5, $6, $7, $8, 'pending', $9)
            "#,
            notif_id,
            notif_req.user_id,
            channel_str,
            notif_req.recipient,
            notif_req.subject,
            notif_req.content.as_deref().unwrap_or(""),
            notif_req.template_id,
            notif_req.template_data,
            notif_req.metadata
        )
        .execute(&mut *tx)
        .await;

        results.push(serde_json::json!({
            "recipient": notif_req.recipient,
            "status": if insert_result.is_ok() { "queued" } else { "failed" },
            "notification_id": if insert_result.is_ok() { Some(notif_id.to_string()) } else { None },
        }));
    }

    tx.commit().await?;

    Ok(HttpResponse::Created().json(serde_json::json!({
        "total": body.notifications.len(),
        "queued": results.iter().filter(|r| r["status"] == "queued").count(),
        "failed": results.iter().filter(|r| r["status"] == "failed").count(),
        "results": results,
    })))
}

pub async fn get_notification_status(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    require_auth(&req)?;
    let notif_id = path.into_inner();

    let notif = sqlx::query!(
        r#"
        SELECT id, status::text, channel::text, recipient, subject,
               sent_at, delivered_at, retry_count, error_message, created_at
        FROM notifications WHERE id = $1
        "#,
        notif_id
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Notification not found".to_string()))?;

    Ok(HttpResponse::Ok().json(notif))
}

pub async fn unsubscribe(
    pool: web::Data<PgPool>,
    body: web::Json<crate::models::preference::UnsubscribeRequest>,
) -> Result<HttpResponse, AppError> {
    // Handle unsubscribe - no auth required for email unsubscribes
    if let Some(channel) = &body.channel {
        match channel.as_str() {
            "email" => {
                sqlx::query!(
                    "UPDATE notification_preferences SET email_enabled = false WHERE user_id = $1",
                    body.user_id
                )
                .execute(pool.get_ref())
                .await?;
            }
            "sms" => {
                sqlx::query!(
                    "UPDATE notification_preferences SET sms_enabled = false WHERE user_id = $1",
                    body.user_id
                )
                .execute(pool.get_ref())
                .await?;
            }
            _ => {}
        }
    } else {
        // Unsubscribe from all channels
        sqlx::query!(
            r#"
            UPDATE notification_preferences
            SET email_enabled = false, sms_enabled = false, push_enabled = false
            WHERE user_id = $1
            "#,
            body.user_id
        )
        .execute(pool.get_ref())
        .await?;
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Unsubscribed successfully"
    })))
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

mod channels;
mod errors;
mod handlers;
mod middleware;
mod models;
mod services;
mod workers;

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

    // Initialize channels
    let email_channel = channels::email::EmailChannel::new(
        &env::var("SMTP_HOST").unwrap_or_else(|_| "localhost".to_string()),
        env::var("SMTP_PORT").ok().and_then(|p| p.parse().ok()).unwrap_or(587),
        &env::var("SMTP_USER").unwrap_or_default(),
        &env::var("SMTP_PASS").unwrap_or_default(),
        &env::var("FROM_EMAIL").unwrap_or_else(|_| "noreply@example.com".to_string()),
        &env::var("FROM_NAME").unwrap_or_else(|_| "Notification Service".to_string()),
    ).expect("Failed to create email channel");

    let sms_channel = channels::sms::SmsChannel::new(
        env::var("TWILIO_ACCOUNT_SID").unwrap_or_else(|_| "MOCK_SID".to_string()),
        env::var("TWILIO_AUTH_TOKEN").unwrap_or_default(),
        env::var("TWILIO_FROM").unwrap_or_else(|_| "+1234567890".to_string()),
    );

    let push_channel = channels::push::PushChannel::new(
        env::var("FCM_SERVER_KEY").unwrap_or_else(|_| "MOCK_KEY".to_string()),
    );

    // Start delivery worker
    let worker = workers::delivery::DeliveryWorker::new(
        pool.clone(),
        email_channel,
        sms_channel,
        push_channel,
    );

    tokio::spawn(async move {
        worker.start().await;
    });

    log::info!("Starting Notification Service at http://{}:{}", host, port);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .app_data(web::Data::new(pool.clone()))
            .service(
                web::scope("/api")
                    .route("/notifications/send", web::post().to(handlers::notifications::send_notification))
                    .route("/notifications/batch", web::post().to(handlers::notifications::send_batch))
                    .route("/notifications/{id}", web::get().to(handlers::notifications::get_notification_status))
                    .route("/notifications/unsubscribe", web::post().to(handlers::notifications::unsubscribe))
                    .route("/templates", web::get().to(handlers::templates::list_templates))
                    .route("/templates", web::post().to(handlers::templates::create_template))
                    .route("/preferences/{user_id}", web::get().to(handlers::preferences::get_preferences))
                    .route("/preferences/{user_id}", web::put().to(handlers::preferences::update_preferences))
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
CREATE TYPE notification_channel AS ENUM ('email', 'sms', 'push', 'websocket', 'webhook');
CREATE TYPE notification_status AS ENUM ('pending', 'queued', 'sending', 'sent', 'delivered', 'failed', 'bounced', 'unsubscribed');
CREATE TYPE notification_priority AS ENUM ('low', 'normal', 'high', 'critical');

CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    channel notification_channel NOT NULL,
    recipient VARCHAR(255) NOT NULL,
    subject VARCHAR(500),
    content TEXT NOT NULL,
    html_content TEXT,
    template_id UUID,
    template_data JSONB,
    status notification_status NOT NULL DEFAULT 'pending',
    priority notification_priority NOT NULL DEFAULT 'normal',
    scheduled_at TIMESTAMPTZ,
    sent_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    retry_count INTEGER NOT NULL DEFAULT 0,
    max_retries INTEGER NOT NULL DEFAULT 3,
    next_retry_at TIMESTAMPTZ,
    error_message TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifs_status ON notifications(status, scheduled_at);
CREATE INDEX idx_notifs_user ON notifications(user_id);
CREATE INDEX idx_notifs_priority ON notifications(priority, created_at);

CREATE TABLE notification_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(110) UNIQUE NOT NULL,
    channel notification_channel NOT NULL,
    subject_template TEXT,
    body_template TEXT NOT NULL,
    html_template TEXT,
    variables TEXT[] NOT NULL DEFAULT '{}',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE notification_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    notification_type VARCHAR(50) NOT NULL DEFAULT 'all',
    email_enabled BOOLEAN NOT NULL DEFAULT true,
    sms_enabled BOOLEAN NOT NULL DEFAULT false,
    push_enabled BOOLEAN NOT NULL DEFAULT true,
    websocket_enabled BOOLEAN NOT NULL DEFAULT true,
    email_address VARCHAR(255),
    phone_number VARCHAR(20),
    push_token TEXT,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## สรุป Part 086

ใน Part นี้เราได้สร้าง Notification Service ที่สมบูรณ์ด้วย:
1. **Multi-channel** รองรับ Email, SMS, Push, WebSocket
2. **Template Engine** ด้วย Tera templating
3. **Delivery Worker** แบบ background processing
4. **Retry Mechanism** ด้วย exponential backoff
5. **Batch Notifications** สำหรับส่ง bulk messages
6. **User Preferences** และ Unsubscribe handling
7. **Delivery Tracking** พร้อม status updates

ใน **Part 087** เราจะสร้าง **Analytics Dashboard API** พร้อม event tracking

---

*[← Part 085: Task Management API](../part_085/README.md) | [Part 087: Analytics Dashboard API →](../part_087/README.md)*
