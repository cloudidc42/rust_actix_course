# Part 099: Capstone Project Part 2 - Social Platform

## บทนำ (Introduction)

ใน Part 2 นี้เราจะเพิ่ม features ขั้นสูงให้กับ Social Platform ได้แก่ Notifications, Email integration, Background jobs, Rate limiting, Analytics, Admin panel, Performance optimization และ Docker/CI-CD setup

## 1. Notifications System

### Real-time notifications

```rust
// src/domain/notification.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "notification_type", rename_all = "snake_case")]
pub enum NotificationType {
    NewFollower,
    PostLike,
    PostComment,
    CommentLike,
    CommentReply,
    Mention,
    NewPost,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Notification {
    pub id: Uuid,
    pub user_id: Uuid,          // recipient
    pub actor_id: Uuid,         // who triggered it
    pub notification_type: NotificationType,
    pub entity_id: Option<Uuid>,// post_id, comment_id, etc.
    pub message: String,
    pub is_read: bool,
    pub created_at: DateTime<Utc>,
}

// src/services/notification_service.rs
use crate::ws::server::{SendToUser, WsServer};
use actix::Addr;

pub struct NotificationService {
    pool: sqlx::PgPool,
    ws_server: Addr<WsServer>,
}

impl NotificationService {
    pub fn new(pool: sqlx::PgPool, ws_server: Addr<WsServer>) -> Self {
        NotificationService { pool, ws_server }
    }
    
    pub async fn create_notification(
        &self,
        user_id: Uuid,
        actor_id: Uuid,
        notification_type: NotificationType,
        entity_id: Option<Uuid>,
        message: &str,
    ) -> Result<Notification, sqlx::Error> {
        let notification = sqlx::query_as!(
            Notification,
            r#"
            INSERT INTO notifications (user_id, actor_id, notification_type, entity_id, message)
            VALUES ($1, $2, $3, $4, $5)
            RETURNING *
            "#,
            user_id,
            actor_id,
            notification_type as NotificationType,
            entity_id,
            message
        )
        .fetch_one(&self.pool)
        .await?;
        
        // Send real-time notification
        let payload = serde_json::json!({
            "type": "notification",
            "data": notification
        });
        
        self.ws_server.do_send(SendToUser {
            user_id,
            message: payload.to_string(),
        });
        
        Ok(notification)
    }
    
    pub async fn notify_new_follower(
        &self,
        followed_id: Uuid,
        follower_id: Uuid,
        follower_username: &str,
    ) -> Result<(), sqlx::Error> {
        self.create_notification(
            followed_id,
            follower_id,
            NotificationType::NewFollower,
            None,
            &format!("{} started following you", follower_username),
        ).await?;
        
        Ok(())
    }
    
    pub async fn notify_post_like(
        &self,
        post_author_id: Uuid,
        liker_id: Uuid,
        post_id: Uuid,
        liker_username: &str,
    ) -> Result<(), sqlx::Error> {
        self.create_notification(
            post_author_id,
            liker_id,
            NotificationType::PostLike,
            Some(post_id),
            &format!("{} liked your post", liker_username),
        ).await?;
        
        Ok(())
    }
    
    pub async fn get_user_notifications(
        &self,
        user_id: Uuid,
        limit: i64,
        offset: i64,
    ) -> Result<Vec<Notification>, sqlx::Error> {
        sqlx::query_as!(
            Notification,
            r#"
            SELECT * FROM notifications
            WHERE user_id = $1
            ORDER BY created_at DESC
            LIMIT $2 OFFSET $3
            "#,
            user_id, limit, offset
        )
        .fetch_all(&self.pool)
        .await
    }
    
    pub async fn mark_as_read(
        &self,
        user_id: Uuid,
        notification_ids: &[Uuid],
    ) -> Result<u64, sqlx::Error> {
        let result = sqlx::query!(
            r#"
            UPDATE notifications
            SET is_read = true
            WHERE user_id = $1 AND id = ANY($2)
            "#,
            user_id,
            notification_ids
        )
        .execute(&self.pool)
        .await?;
        
        Ok(result.rows_affected())
    }
    
    pub async fn get_unread_count(&self, user_id: Uuid) -> Result<i64, sqlx::Error> {
        let row = sqlx::query!(
            "SELECT COUNT(*) FROM notifications WHERE user_id = $1 AND is_read = false",
            user_id
        )
        .fetch_one(&self.pool)
        .await?;
        
        Ok(row.count.unwrap_or(0))
    }
}
```

## 2. Email Integration

### Lettre email service

```rust
// src/services/email_service.rs
use lettre::{
    transport::smtp::authentication::Credentials,
    Message, SmtpTransport, Transport,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone)]
pub struct EmailConfig {
    pub smtp_host: String,
    pub smtp_port: u16,
    pub username: String,
    pub password: String,
    pub from_email: String,
    pub from_name: String,
    pub base_url: String,
}

pub struct EmailService {
    transport: SmtpTransport,
    config: EmailConfig,
}

impl EmailService {
    pub fn new(config: EmailConfig) -> Result<Self, anyhow::Error> {
        let credentials = Credentials::new(
            config.username.clone(),
            config.password.clone(),
        );
        
        let transport = SmtpTransport::relay(&config.smtp_host)?
            .port(config.smtp_port)
            .credentials(credentials)
            .build();
        
        Ok(EmailService { transport, config })
    }
    
    pub async fn send_welcome_email(
        &self,
        to_email: &str,
        username: &str,
    ) -> Result<(), anyhow::Error> {
        let html = format!(r#"
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="utf-8">
            <title>Welcome to Social Platform</title>
            <style>
                body {{ font-family: Arial, sans-serif; }}
                .container {{ max-width: 600px; margin: 0 auto; padding: 20px; }}
                .header {{ background: #1DA1F2; color: white; padding: 20px; }}
                .button {{ 
                    background: #1DA1F2; color: white; padding: 10px 20px;
                    text-decoration: none; border-radius: 5px; display: inline-block;
                }}
            </style>
        </head>
        <body>
            <div class="container">
                <div class="header">
                    <h1>Welcome to Social Platform!</h1>
                </div>
                <p>Hi <strong>{}</strong>,</p>
                <p>Thank you for joining our community! Your account has been created successfully.</p>
                <p>Get started by:</p>
                <ul>
                    <li>Setting up your profile</li>
                    <li>Finding people to follow</li>
                    <li>Creating your first post</li>
                </ul>
                <a href="{}/profile" class="button">Visit Your Profile</a>
                <p>Happy posting!</p>
            </div>
        </body>
        </html>
        "#, username, self.config.base_url);
        
        self.send_email(
            to_email,
            &format!("Welcome to Social Platform, {}!", username),
            &html,
        ).await
    }
    
    pub async fn send_password_reset_email(
        &self,
        to_email: &str,
        username: &str,
        reset_token: &str,
    ) -> Result<(), anyhow::Error> {
        let reset_url = format!("{}/reset-password?token={}", self.config.base_url, reset_token);
        
        let html = format!(r#"
        <!DOCTYPE html>
        <html>
        <body>
            <h2>Password Reset Request</h2>
            <p>Hi {},</p>
            <p>You requested to reset your password. Click the link below to continue:</p>
            <a href="{}">Reset Password</a>
            <p>This link expires in 1 hour.</p>
            <p>If you didn't request this, please ignore this email.</p>
        </body>
        </html>
        "#, username, reset_url);
        
        self.send_email(
            to_email,
            "Reset Your Password",
            &html,
        ).await
    }
    
    pub async fn send_notification_digest(
        &self,
        to_email: &str,
        username: &str,
        notifications: &[String],
    ) -> Result<(), anyhow::Error> {
        let items = notifications.iter()
            .map(|n| format!("<li>{}</li>", n))
            .collect::<Vec<_>>()
            .join("\n");
        
        let html = format!(r#"
        <html>
        <body>
            <h2>Your Daily Notification Digest</h2>
            <p>Hi {},</p>
            <p>Here's what happened today:</p>
            <ul>{}</ul>
            <a href="{}/notifications">View All Notifications</a>
        </body>
        </html>
        "#, username, items, self.config.base_url);
        
        self.send_email(
            to_email,
            "Your Daily Digest",
            &html,
        ).await
    }
    
    async fn send_email(
        &self,
        to: &str,
        subject: &str,
        html: &str,
    ) -> Result<(), anyhow::Error> {
        let message = Message::builder()
            .from(format!("{} <{}>", self.config.from_name, self.config.from_email).parse()?)
            .to(to.parse()?)
            .subject(subject)
            .header(lettre::message::header::ContentType::TEXT_HTML)
            .body(html.to_string())?;
        
        tokio::task::spawn_blocking({
            let transport = self.transport.clone();
            let message = message;
            move || transport.send(&message)
        })
        .await??;
        
        Ok(())
    }
}
```

## 3. Background Jobs (Image Processing)

### Tokio task queue สำหรับ background jobs

```rust
// src/jobs/mod.rs
use tokio::sync::mpsc;
use std::sync::Arc;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Job {
    ProcessImage {
        user_id: uuid::Uuid,
        file_path: String,
        job_type: ImageJobType,
    },
    SendEmail {
        to: String,
        subject: String,
        body: String,
    },
    SendNotification {
        user_id: uuid::Uuid,
        message: String,
    },
    UpdateUserStats {
        user_id: uuid::Uuid,
    },
    GenerateThumbnail {
        post_id: uuid::Uuid,
        image_url: String,
    },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum ImageJobType {
    Avatar,
    Cover,
    PostMedia,
}

pub struct JobQueue {
    sender: mpsc::Sender<Job>,
}

impl JobQueue {
    pub fn new() -> (Self, JobWorker) {
        let (tx, rx) = mpsc::channel(1000);
        
        let queue = JobQueue { sender: tx };
        let worker = JobWorker { receiver: rx };
        
        (queue, worker)
    }
    
    pub async fn enqueue(&self, job: Job) -> Result<(), anyhow::Error> {
        self.sender.send(job).await
            .map_err(|e| anyhow::anyhow!("Failed to enqueue job: {}", e))?;
        Ok(())
    }
}

pub struct JobWorker {
    receiver: mpsc::Receiver<Job>,
}

impl JobWorker {
    pub async fn run(mut self) {
        println!("Job worker started");
        
        while let Some(job) = self.receiver.recv().await {
            tokio::spawn(process_job(job));
        }
        
        println!("Job worker stopped");
    }
}

async fn process_job(job: Job) {
    match job {
        Job::ProcessImage { user_id, file_path, job_type } => {
            process_image(user_id, &file_path, job_type).await;
        }
        Job::SendEmail { to, subject, body } => {
            println!("Sending email to {}: {}", to, subject);
            // email sending logic
        }
        Job::SendNotification { user_id, message } => {
            println!("Sending notification to {}: {}", user_id, message);
        }
        Job::UpdateUserStats { user_id } => {
            println!("Updating stats for user {}", user_id);
        }
        Job::GenerateThumbnail { post_id, image_url } => {
            println!("Generating thumbnail for post {}: {}", post_id, image_url);
        }
    }
}

async fn process_image(user_id: uuid::Uuid, file_path: &str, job_type: ImageJobType) {
    use image::{GenericImageView, imageops::FilterType};
    
    println!("Processing image: {}", file_path);
    
    match tokio::fs::read(file_path).await {
        Ok(data) => {
            match image::load_from_memory(&data) {
                Ok(img) => {
                    let (width, height) = match job_type {
                        ImageJobType::Avatar => (200, 200),
                        ImageJobType::Cover => (1500, 500),
                        ImageJobType::PostMedia => (1200, 1200),
                    };
                    
                    let resized = img.resize_to_fill(width, height, FilterType::Lanczos3);
                    
                    let output_path = format!("{}.processed.webp", file_path);
                    if let Err(e) = resized.save(&output_path) {
                        eprintln!("Failed to save processed image: {}", e);
                    } else {
                        println!("Image processed: {}", output_path);
                    }
                }
                Err(e) => eprintln!("Failed to load image {}: {}", file_path, e),
            }
        }
        Err(e) => eprintln!("Failed to read file {}: {}", file_path, e),
    }
}
```

## 4. Rate Limiting

### Token bucket rate limiter

```rust
// src/middleware/rate_limit.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use std::{
    collections::HashMap,
    future::{ready, Ready},
    sync::{Arc, Mutex},
    time::{Duration, Instant},
};

#[derive(Clone)]
struct TokenBucket {
    tokens: f64,
    max_tokens: f64,
    refill_rate: f64, // tokens per second
    last_refill: Instant,
}

impl TokenBucket {
    fn new(max_tokens: f64, refill_rate: f64) -> Self {
        TokenBucket {
            tokens: max_tokens,
            max_tokens,
            refill_rate,
            last_refill: Instant::now(),
        }
    }
    
    fn try_consume(&mut self, tokens: f64) -> bool {
        self.refill();
        
        if self.tokens >= tokens {
            self.tokens -= tokens;
            true
        } else {
            false
        }
    }
    
    fn refill(&mut self) {
        let now = Instant::now();
        let elapsed = now.duration_since(self.last_refill).as_secs_f64();
        self.tokens = (self.tokens + elapsed * self.refill_rate).min(self.max_tokens);
        self.last_refill = now;
    }
}

type BucketMap = Arc<Mutex<HashMap<String, TokenBucket>>>;

pub struct RateLimitConfig {
    pub max_requests: f64,
    pub window_seconds: f64,
}

pub struct RateLimiter {
    buckets: BucketMap,
    config: RateLimitConfig,
}

impl RateLimiter {
    pub fn new(max_requests: f64, window_seconds: f64) -> Self {
        RateLimiter {
            buckets: Arc::new(Mutex::new(HashMap::new())),
            config: RateLimitConfig { max_requests, window_seconds },
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimiter
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RateLimiterMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimiterMiddleware {
            service: Arc::new(service),
            buckets: self.buckets.clone(),
            max_requests: self.config.max_requests,
            refill_rate: self.config.max_requests / self.config.window_seconds,
        }))
    }
}

pub struct RateLimiterMiddleware<S> {
    service: Arc<S>,
    buckets: BucketMap,
    max_requests: f64,
    refill_rate: f64,
}

impl<S, B> Service<ServiceRequest> for RateLimiterMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    S::Future: 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = std::pin::Pin<Box<dyn std::future::Future<Output = Result<Self::Response, Self::Error>>>>;
    
    forward_ready!(service);
    
    fn call(&self, req: ServiceRequest) -> Self::Future {
        let client_ip = req.peer_addr()
            .map(|addr| addr.ip().to_string())
            .unwrap_or_else(|| "unknown".to_string());
        
        let mut buckets = self.buckets.lock().unwrap();
        
        let bucket = buckets.entry(client_ip.clone())
            .or_insert_with(|| TokenBucket::new(self.max_requests, self.refill_rate));
        
        let allowed = bucket.try_consume(1.0);
        drop(buckets);
        
        if !allowed {
            let response = HttpResponse::TooManyRequests()
                .append_header(("X-RateLimit-Limit", self.max_requests.to_string()))
                .json(serde_json::json!({
                    "error": "Rate limit exceeded",
                    "message": "Too many requests, please try again later"
                }));
            
            return Box::pin(async move {
                Ok(req.into_response(response.map_into_right_body()))
            });
        }
        
        let service = self.service.clone();
        let max_requests = self.max_requests;
        
        Box::pin(async move {
            let mut res = service.call(req).await?;
            res.headers_mut().insert(
                actix_web::http::header::HeaderName::from_static("x-ratelimit-limit"),
                actix_web::http::header::HeaderValue::from_str(&max_requests.to_string()).unwrap(),
            );
            Ok(res)
        })
    }
}
```

## 5. Analytics

### User and post analytics

```rust
// src/services/analytics_service.rs
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc, Duration};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize)]
pub struct UserAnalytics {
    pub user_id: Uuid,
    pub period: String,
    pub new_followers: i64,
    pub post_views: i64,
    pub post_likes: i64,
    pub profile_views: i64,
    pub engagement_rate: f64,
    pub top_posts: Vec<PostStats>,
}

#[derive(Debug, Serialize)]
pub struct PostStats {
    pub post_id: Uuid,
    pub content_preview: String,
    pub views: i64,
    pub likes: i64,
    pub comments: i64,
    pub shares: i64,
    pub engagement_rate: f64,
}

#[derive(Debug, Serialize)]
pub struct PlatformAnalytics {
    pub period: String,
    pub total_users: i64,
    pub new_users: i64,
    pub active_users: i64,
    pub total_posts: i64,
    pub new_posts: i64,
    pub total_engagements: i64,
    pub dau_mau_ratio: f64,
    pub top_hashtags: Vec<HashtagStats>,
}

#[derive(Debug, Serialize)]
pub struct HashtagStats {
    pub hashtag: String,
    pub post_count: i64,
    pub engagement: i64,
}

pub struct AnalyticsService {
    pool: PgPool,
}

impl AnalyticsService {
    pub fn new(pool: PgPool) -> Self {
        AnalyticsService { pool }
    }
    
    pub async fn get_user_analytics(
        &self,
        user_id: Uuid,
        days: i64,
    ) -> Result<UserAnalytics, sqlx::Error> {
        let since = Utc::now() - Duration::days(days);
        
        let new_followers = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM follows WHERE following_id = $1 AND created_at > $2",
            user_id, since
        )
        .fetch_one(&self.pool)
        .await?
        .unwrap_or(0);
        
        let post_likes = sqlx::query_scalar!(
            r#"
            SELECT COALESCE(SUM(p.like_count), 0)
            FROM posts p
            WHERE p.user_id = $1 AND p.created_at > $2
            "#,
            user_id, since
        )
        .fetch_one(&self.pool)
        .await?
        .unwrap_or(0);
        
        let top_posts = sqlx::query!(
            r#"
            SELECT id, content, like_count, comment_count, share_count
            FROM posts
            WHERE user_id = $1 AND created_at > $2 AND is_deleted = false
            ORDER BY like_count + comment_count DESC
            LIMIT 5
            "#,
            user_id, since
        )
        .fetch_all(&self.pool)
        .await?
        .into_iter()
        .map(|row| {
            let total = row.like_count + row.comment_count;
            PostStats {
                post_id: row.id,
                content_preview: row.content.chars().take(100).collect(),
                views: 0,
                likes: row.like_count,
                comments: row.comment_count,
                shares: row.share_count,
                engagement_rate: if total > 0 {
                    (row.like_count + row.comment_count) as f64 / total as f64
                } else { 0.0 },
            }
        })
        .collect();
        
        Ok(UserAnalytics {
            user_id,
            period: format!("{}d", days),
            new_followers,
            post_views: 0,
            post_likes,
            profile_views: 0,
            engagement_rate: 0.0,
            top_posts,
        })
    }
    
    pub async fn get_platform_analytics(
        &self,
        days: i64,
    ) -> Result<PlatformAnalytics, sqlx::Error> {
        let since = Utc::now() - Duration::days(days);
        
        let total_users = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
            .fetch_one(&self.pool).await?.unwrap_or(0);
        
        let new_users = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM users WHERE created_at > $1",
            since
        )
        .fetch_one(&self.pool).await?.unwrap_or(0);
        
        let new_posts = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM posts WHERE created_at > $1 AND is_deleted = false",
            since
        )
        .fetch_one(&self.pool).await?.unwrap_or(0);
        
        let top_hashtags = sqlx::query!(
            r#"
            SELECT unnest(hashtags) as hashtag, COUNT(*) as count
            FROM posts
            WHERE created_at > $1 AND is_deleted = false
            GROUP BY hashtag
            ORDER BY count DESC
            LIMIT 10
            "#,
            since
        )
        .fetch_all(&self.pool)
        .await?
        .into_iter()
        .map(|row| HashtagStats {
            hashtag: row.hashtag.unwrap_or_default(),
            post_count: row.count.unwrap_or(0),
            engagement: 0,
        })
        .collect();
        
        Ok(PlatformAnalytics {
            period: format!("{}d", days),
            total_users,
            new_users,
            active_users: 0,
            total_posts: 0,
            new_posts,
            total_engagements: 0,
            dau_mau_ratio: 0.0,
            top_hashtags,
        })
    }
}
```

## 6. Admin Panel

### Admin API handlers

```rust
// src/handlers/admin.rs
use actix_web::{web, HttpResponse};
use uuid::Uuid;
use crate::{
    auth::jwt::Claims,
    services::analytics_service::AnalyticsService,
    errors::AppError,
};

pub async fn get_dashboard(
    db: web::Data<sqlx::PgPool>,
    claims: web::ReqData<Claims>,
) -> Result<HttpResponse, AppError> {
    require_admin(&claims)?;
    
    let analytics = AnalyticsService::new(db.get_ref().clone());
    let stats = analytics.get_platform_analytics(30).await?;
    
    Ok(HttpResponse::Ok().json(stats))
}

pub async fn list_users_admin(
    db: web::Data<sqlx::PgPool>,
    claims: web::ReqData<Claims>,
    query: web::Query<AdminUserQuery>,
) -> Result<HttpResponse, AppError> {
    require_admin(&claims)?;
    
    let page = query.page.unwrap_or(1);
    let limit = query.limit.unwrap_or(50).min(100);
    let offset = (page - 1) * limit;
    
    let users = sqlx::query!(
        r#"
        SELECT id, username, email, is_verified, is_active, 
               follower_count, post_count, created_at
        FROM users
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
        "#,
        limit as i64,
        offset as i64
    )
    .fetch_all(db.get_ref())
    .await?;
    
    let total = sqlx::query_scalar!("SELECT COUNT(*) FROM users")
        .fetch_one(db.get_ref())
        .await?
        .unwrap_or(0);
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "users": users.iter().map(|u| serde_json::json!({
            "id": u.id,
            "username": u.username,
            "email": u.email,
            "is_verified": u.is_verified,
            "is_active": u.is_active,
            "follower_count": u.follower_count,
            "post_count": u.post_count,
            "created_at": u.created_at
        })).collect::<Vec<_>>(),
        "total": total,
        "page": page,
        "limit": limit
    })))
}

pub async fn ban_user(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
    body: web::Json<BanRequest>,
) -> Result<HttpResponse, AppError> {
    require_admin(&claims)?;
    
    let user_id = path.into_inner();
    
    sqlx::query!(
        "UPDATE users SET is_active = false WHERE id = $1",
        user_id
    )
    .execute(db.get_ref())
    .await?;
    
    // Log admin action
    sqlx::query!(
        r#"
        INSERT INTO admin_logs (admin_id, action, target_id, reason)
        VALUES ($1, 'ban_user', $2, $3)
        "#,
        claims.sub.parse::<Uuid>()?,
        user_id,
        body.reason
    )
    .execute(db.get_ref())
    .await?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "message": format!("User {} has been banned", user_id)
    })))
}

pub async fn delete_post_admin(
    db: web::Data<sqlx::PgPool>,
    path: web::Path<Uuid>,
    claims: web::ReqData<Claims>,
    body: web::Json<ModerateRequest>,
) -> Result<HttpResponse, AppError> {
    require_admin(&claims)?;
    
    let post_id = path.into_inner();
    
    sqlx::query!(
        "UPDATE posts SET is_deleted = true WHERE id = $1",
        post_id
    )
    .execute(db.get_ref())
    .await?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({"success": true})))
}

#[derive(serde::Deserialize)]
pub struct AdminUserQuery {
    page: Option<u64>,
    limit: Option<u64>,
}

#[derive(serde::Deserialize)]
pub struct BanRequest {
    reason: String,
}

#[derive(serde::Deserialize)]
pub struct ModerateRequest {
    reason: String,
}

fn require_admin(claims: &Claims) -> Result<(), AppError> {
    // Check admin role from JWT claims or database
    // Simplified check
    Ok(())
}
```

## 7. Performance Optimization

### Caching ด้วย Redis

```rust
// src/services/cache_service.rs
use redis::AsyncCommands;
use serde::{de::DeserializeOwned, Serialize};
use std::time::Duration;

pub struct CacheService {
    client: redis::Client,
    default_ttl: Duration,
}

impl CacheService {
    pub fn new(redis_url: &str) -> Result<Self, redis::RedisError> {
        let client = redis::Client::open(redis_url)?;
        Ok(CacheService {
            client,
            default_ttl: Duration::from_secs(300), // 5 minutes
        })
    }
    
    pub async fn get<T: DeserializeOwned>(
        &self,
        key: &str,
    ) -> Option<T> {
        let mut conn = self.client.get_async_connection().await.ok()?;
        let data: Option<String> = conn.get(key).await.ok()?;
        data.and_then(|d| serde_json::from_str(&d).ok())
    }
    
    pub async fn set<T: Serialize>(
        &self,
        key: &str,
        value: &T,
        ttl: Option<Duration>,
    ) -> Result<(), anyhow::Error> {
        let mut conn = self.client.get_async_connection().await?;
        let data = serde_json::to_string(value)?;
        let ttl_secs = ttl.unwrap_or(self.default_ttl).as_secs() as usize;
        conn.set_ex(key, data, ttl_secs).await?;
        Ok(())
    }
    
    pub async fn delete(&self, key: &str) -> Result<(), anyhow::Error> {
        let mut conn = self.client.get_async_connection().await?;
        conn.del(key).await?;
        Ok(())
    }
    
    pub async fn increment(&self, key: &str) -> Result<i64, anyhow::Error> {
        let mut conn = self.client.get_async_connection().await?;
        Ok(conn.incr(key, 1i64).await?)
    }
    
    // Cache-aside pattern
    pub async fn get_or_set<T, F, Fut>(
        &self,
        key: &str,
        ttl: Option<Duration>,
        loader: F,
    ) -> Result<T, anyhow::Error>
    where
        T: Serialize + DeserializeOwned,
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = Result<T, anyhow::Error>>,
    {
        if let Some(cached) = self.get::<T>(key).await {
            return Ok(cached);
        }
        
        let value = loader().await?;
        self.set(key, &value, ttl).await?;
        Ok(value)
    }
    
    // Feed caching with sorted sets
    pub async fn cache_feed(
        &self,
        user_id: &str,
        post_ids: &[(f64, &str)], // (score, post_id)
    ) -> Result<(), anyhow::Error> {
        let mut conn = self.client.get_async_connection().await?;
        let key = format!("feed:{}", user_id);
        
        for (score, post_id) in post_ids {
            conn.zadd::<_, _, _, ()>(&key, score, post_id).await?;
        }
        
        // Keep only top 1000 posts in feed cache
        conn.zremrangebyrank::<_, ()>(&key, 0, -1001).await?;
        conn.expire::<_, ()>(&key, 3600).await?; // 1 hour
        
        Ok(())
    }
    
    pub async fn get_cached_feed(
        &self,
        user_id: &str,
        offset: i64,
        limit: i64,
    ) -> Result<Vec<String>, anyhow::Error> {
        let mut conn = self.client.get_async_connection().await?;
        let key = format!("feed:{}", user_id);
        
        let post_ids: Vec<String> = conn
            .zrevrange(key, offset, offset + limit - 1)
            .await?;
        
        Ok(post_ids)
    }
}
```

## 8. Docker & CI/CD

### Docker setup

```dockerfile
# Dockerfile
FROM rust:1.75-slim-bookworm as builder

# Install system dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Cache dependencies
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm src/main.rs

# Build application
COPY . .
RUN cargo build --release

# Runtime image
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    libssl3 \
    libpq5 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY --from=builder /app/target/release/social-platform /app/
COPY --from=builder /app/migrations /app/migrations
COPY --from=builder /app/static /app/static

RUN mkdir -p /app/uploads

ENV RUST_LOG=info

EXPOSE 8080

CMD ["/app/social-platform"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://postgres:password@db:5432/social_platform
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET:-development-secret-change-in-production}
      UPLOAD_DIR: /app/uploads
    volumes:
      - uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    restart: unless-stopped
  
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: social_platform
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - uploads:/app/uploads:ro
    depends_on:
      - app

volumes:
  postgres_data:
  redis_data:
  uploads:
```

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          components: clippy, rustfmt
      
      - name: Cache cargo
        uses: actions/cache@v3
        with:
          path: |
            ~/.cargo/registry
            ~/.cargo/git
            target
          key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
      
      - name: Check formatting
        run: cargo fmt --all -- --check
      
      - name: Clippy
        run: cargo clippy -- -D warnings
      
      - name: Run migrations
        run: cargo sqlx migrate run
      
      - name: Run tests
        run: cargo test --all
        env:
          RUST_LOG: debug
  
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t social-platform:${{ github.sha }} .
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Push image
        run: |
          docker tag social-platform:${{ github.sha }} \
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            cd /opt/social-platform
            docker pull ghcr.io/${{ github.repository }}:${{ github.sha }}
            docker-compose up -d --no-deps app
            docker system prune -f
```

## สรุป (Summary)

ใน Part 2 ของ Capstone Project เราได้เพิ่ม:
- **Notifications**: Real-time notifications ด้วย WebSocket
- **Email**: Welcome email, password reset ด้วย Lettre
- **Background Jobs**: Async job queue สำหรับ image processing
- **Rate Limiting**: Token bucket algorithm
- **Analytics**: User และ platform analytics
- **Admin Panel**: User management, moderation tools
- **Performance**: Redis caching, feed optimization
- **Docker**: Complete containerization
- **CI/CD**: GitHub Actions pipeline

---

[← Part 098](../part_098/README.md) | [Part 100 →](../part_100/README.md)
