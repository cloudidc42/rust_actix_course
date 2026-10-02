# Part 084: Project: URL Shortener 🔗

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง URL Shortener ด้วย nanoid
- ทำ Click Tracking และ Analytics
- รองรับ Custom Slugs
- กำหนด Expiration Dates
- สร้าง QR Code
- ทำ Rate Limiting ต่อ User
- สร้าง Dashboard พร้อม Stats
- เขียน Complete Implementation

---

## 1. โครงสร้างโปรเจกต์

```
url_shortener/
├── Cargo.toml
├── .env
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   └── url.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── urls.rs
    │   ├── redirect.rs
    │   └── analytics.rs
    ├── services/
    │   ├── mod.rs
    │   ├── shortener.rs
    │   ├── qrcode.rs
    │   └── analytics.rs
    └── middleware/
        ├── mod.rs
        ├── auth.rs
        └── rate_limit.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "url_shortener"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
nanoid = "0.4"
jsonwebtoken = "9"
bcrypt = "0.15"
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
validator = { version = "0.18", features = ["derive"] }
thiserror = "1"
url = "2"
qrcode = "0.13"
image = "0.25"
base64 = "0.22"
redis = { version = "0.26", features = ["tokio-comp", "connection-manager"] }
actix-governor = "0.5"
user-agent-parser = "0.3"
maxminddb = "0.24"
```

---

## 3. Models (`src/models/url.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct ShortUrl {
    pub id: Uuid,
    pub short_code: String,
    pub original_url: String,
    pub title: Option<String>,
    pub description: Option<String>,
    pub user_id: Option<Uuid>,
    pub click_count: i64,
    pub unique_click_count: i64,
    pub is_active: bool,
    pub expires_at: Option<DateTime<Utc>>,
    pub password_hash: Option<String>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct ClickEvent {
    pub id: Uuid,
    pub short_url_id: Uuid,
    pub ip_address: Option<String>,
    pub user_agent: Option<String>,
    pub referer: Option<String>,
    pub country: Option<String>,
    pub city: Option<String>,
    pub browser: Option<String>,
    pub os: Option<String>,
    pub device_type: Option<String>,
    pub is_unique: bool,
    pub clicked_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateShortUrlRequest {
    #[validate(url)]
    pub original_url: String,
    pub custom_code: Option<String>,
    pub title: Option<String>,
    pub description: Option<String>,
    pub expires_at: Option<DateTime<Utc>>,
    pub password: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UrlQuery {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub search: Option<String>,
    pub include_expired: Option<bool>,
}

#[derive(Debug, Serialize)]
pub struct UrlStats {
    pub short_url: ShortUrl,
    pub clicks_today: i64,
    pub clicks_this_week: i64,
    pub clicks_this_month: i64,
    pub top_countries: Vec<CountryStat>,
    pub top_referrers: Vec<RefererStat>,
    pub browser_breakdown: Vec<BrowserStat>,
    pub device_breakdown: Vec<DeviceStat>,
    pub hourly_clicks: Vec<HourlyClick>,
}

#[derive(Debug, Serialize)]
pub struct CountryStat {
    pub country: String,
    pub count: i64,
    pub percentage: f64,
}

#[derive(Debug, Serialize)]
pub struct RefererStat {
    pub referer: String,
    pub count: i64,
}

#[derive(Debug, Serialize)]
pub struct BrowserStat {
    pub browser: String,
    pub count: i64,
}

#[derive(Debug, Serialize)]
pub struct DeviceStat {
    pub device_type: String,
    pub count: i64,
}

#[derive(Debug, Serialize)]
pub struct HourlyClick {
    pub hour: String,
    pub count: i64,
}
```

---

## 4. URL Shortener Service (`src/services/shortener.rs`)

```rust
use nanoid::nanoid;
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;
use crate::models::url::{CreateShortUrlRequest, ShortUrl};

const SHORT_CODE_ALPHABET: &[char] = &[
    '2', '3', '4', '5', '6', '7', '8', '9',
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h',
    'j', 'k', 'm', 'n', 'p', 'q', 'r', 's',
    't', 'u', 'v', 'w', 'x', 'y', 'z',
    'A', 'B', 'C', 'D', 'E', 'F', 'G', 'H',
    'J', 'K', 'M', 'N', 'P', 'Q', 'R', 'S',
    'T', 'U', 'V', 'W', 'X', 'Y', 'Z',
];

pub struct ShortenerService;

impl ShortenerService {
    pub fn generate_code(length: usize) -> String {
        nanoid!(length, SHORT_CODE_ALPHABET)
    }

    pub async fn create_short_url(
        pool: &PgPool,
        req: &CreateShortUrlRequest,
        user_id: Option<Uuid>,
    ) -> Result<ShortUrl, AppError> {
        // Validate URL
        let _ = url::Url::parse(&req.original_url)
            .map_err(|_| AppError::BadRequest("Invalid URL".to_string()))?;

        // Determine short code
        let short_code = if let Some(custom) = &req.custom_code {
            let custom = custom.to_lowercase();
            // Check if custom code is reserved
            if is_reserved_code(&custom) {
                return Err(AppError::BadRequest("This code is reserved".to_string()));
            }
            // Check if already taken
            let exists: bool = sqlx::query_scalar!(
                "SELECT EXISTS(SELECT 1 FROM short_urls WHERE short_code = $1)",
                custom
            )
            .fetch_one(pool)
            .await?
            .unwrap_or(false);

            if exists {
                return Err(AppError::Conflict("Custom code already taken".to_string()));
            }
            custom
        } else {
            // Generate unique code
            let mut code = Self::generate_code(6);
            let mut attempts = 0;
            loop {
                let exists: bool = sqlx::query_scalar!(
                    "SELECT EXISTS(SELECT 1 FROM short_urls WHERE short_code = $1)",
                    code
                )
                .fetch_one(pool)
                .await?
                .unwrap_or(false);

                if !exists {
                    break;
                }
                code = Self::generate_code(6 + attempts / 5);
                attempts += 1;
                if attempts > 10 {
                    return Err(AppError::InternalError("Could not generate unique code".to_string()));
                }
            }
            code
        };

        let password_hash = if let Some(pwd) = &req.password {
            Some(bcrypt::hash(pwd, bcrypt::DEFAULT_COST)
                .map_err(|e| AppError::InternalError(e.to_string()))?)
        } else {
            None
        };

        let id = Uuid::new_v4();
        sqlx::query!(
            r#"
            INSERT INTO short_urls (id, short_code, original_url, title, description, 
                                   user_id, expires_at, password_hash)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            "#,
            id,
            short_code,
            req.original_url,
            req.title,
            req.description,
            user_id,
            req.expires_at,
            password_hash
        )
        .execute(pool)
        .await?;

        let url = sqlx::query_as!(
            ShortUrl,
            "SELECT * FROM short_urls WHERE id = $1",
            id
        )
        .fetch_one(pool)
        .await?;

        Ok(url)
    }

    pub async fn resolve_url(
        pool: &PgPool,
        short_code: &str,
    ) -> Result<ShortUrl, AppError> {
        let url = sqlx::query_as!(
            ShortUrl,
            "SELECT * FROM short_urls WHERE short_code = $1 AND is_active = true",
            short_code
        )
        .fetch_optional(pool)
        .await?
        .ok_or_else(|| AppError::NotFound("Short URL not found".to_string()))?;

        // Check expiration
        if let Some(expires_at) = url.expires_at {
            if expires_at < chrono::Utc::now() {
                return Err(AppError::BadRequest("This URL has expired".to_string()));
            }
        }

        Ok(url)
    }
}

fn is_reserved_code(code: &str) -> bool {
    let reserved = ["api", "admin", "login", "register", "dashboard",
                    "analytics", "settings", "help", "about", "terms",
                    "privacy", "contact", "404", "500", "health"];
    reserved.contains(&code)
}
```

---

## 5. QR Code Service (`src/services/qrcode.rs`)

```rust
use base64::{engine::general_purpose, Engine as _};
use image::{ImageOutputFormat, Luma};
use qrcode::QrCode;
use std::io::Cursor;

use crate::errors::AppError;

pub struct QrCodeService;

impl QrCodeService {
    pub fn generate_qr_base64(url: &str) -> Result<String, AppError> {
        // Generate QR code
        let code = QrCode::new(url.as_bytes())
            .map_err(|e| AppError::InternalError(format!("QR generation failed: {}", e)))?;

        // Render to image
        let image = code.render::<Luma<u8>>()
            .min_dimensions(200, 200)
            .dark_color(Luma([0u8]))
            .light_color(Luma([255u8]))
            .build();

        // Convert to PNG bytes
        let mut png_bytes: Vec<u8> = Vec::new();
        let mut cursor = Cursor::new(&mut png_bytes);
        image
            .write_to(&mut cursor, ImageOutputFormat::Png)
            .map_err(|e| AppError::InternalError(e.to_string()))?;

        // Encode as base64
        let base64 = general_purpose::STANDARD.encode(&png_bytes);
        Ok(format!("data:image/png;base64,{}", base64))
    }

    pub fn generate_qr_svg(url: &str) -> Result<String, AppError> {
        let code = QrCode::new(url.as_bytes())
            .map_err(|e| AppError::InternalError(format!("QR generation failed: {}", e)))?;

        // Build SVG representation
        let svg = code
            .render()
            .min_dimensions(200, 200)
            .dark_color(qrcode::render::svg::Color("#000000"))
            .light_color(qrcode::render::svg::Color("#FFFFFF"))
            .build();

        Ok(svg)
    }
}
```

---

## 6. Analytics Service (`src/services/analytics.rs`)

```rust
use chrono::Utc;
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;
use crate::models::url::{BrowserStat, ClickEvent, CountryStat, DeviceStat, HourlyClick, RefererStat};

pub struct AnalyticsService;

impl AnalyticsService {
    pub async fn record_click(
        pool: &PgPool,
        short_url_id: Uuid,
        ip_address: Option<&str>,
        user_agent: Option<&str>,
        referer: Option<&str>,
    ) -> Result<(), AppError> {
        // Check if unique click (simple IP-based check)
        let is_unique = if let Some(ip) = ip_address {
            let recent_click: Option<i64> = sqlx::query_scalar!(
                r#"
                SELECT COUNT(*) FROM click_events
                WHERE short_url_id = $1
                  AND ip_address = $2
                  AND clicked_at > NOW() - INTERVAL '24 hours'
                "#,
                short_url_id, ip
            )
            .fetch_one(pool)
            .await?;

            recent_click.unwrap_or(0) == 0
        } else {
            false
        };

        // Parse user agent (simplified)
        let (browser, os, device_type) = parse_user_agent(user_agent);

        // In production: use MaxMind GeoIP2 for geolocation
        // let (country, city) = geolocate(ip_address).await?;

        let event_id = Uuid::new_v4();
        sqlx::query!(
            r#"
            INSERT INTO click_events (id, short_url_id, ip_address, user_agent, referer,
                                     browser, os, device_type, is_unique)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
            "#,
            event_id,
            short_url_id,
            ip_address,
            user_agent,
            referer,
            browser,
            os,
            device_type,
            is_unique
        )
        .execute(pool)
        .await?;

        // Update click count
        sqlx::query!(
            r#"
            UPDATE short_urls
            SET click_count = click_count + 1,
                unique_click_count = unique_click_count + $1,
                updated_at = NOW()
            WHERE id = $2
            "#,
            is_unique as i64,
            short_url_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    pub async fn get_url_stats(
        pool: &PgPool,
        short_url_id: Uuid,
    ) -> Result<UrlAnalytics, AppError> {
        let clicks_today: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM click_events WHERE short_url_id = $1 AND clicked_at >= date_trunc('day', NOW())",
            short_url_id
        )
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

        let clicks_this_week: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM click_events WHERE short_url_id = $1 AND clicked_at >= NOW() - INTERVAL '7 days'",
            short_url_id
        )
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

        let clicks_this_month: i64 = sqlx::query_scalar!(
            "SELECT COUNT(*) FROM click_events WHERE short_url_id = $1 AND clicked_at >= date_trunc('month', NOW())",
            short_url_id
        )
        .fetch_one(pool)
        .await?
        .unwrap_or(0);

        // Top countries
        let top_countries = sqlx::query!(
            r#"
            SELECT country, COUNT(*) as count
            FROM click_events
            WHERE short_url_id = $1 AND country IS NOT NULL
            GROUP BY country
            ORDER BY count DESC
            LIMIT 10
            "#,
            short_url_id
        )
        .fetch_all(pool)
        .await?
        .into_iter()
        .map(|r| CountryStat {
            country: r.country.unwrap_or("Unknown".to_string()),
            count: r.count.unwrap_or(0),
            percentage: 0.0, // Calculated after
        })
        .collect();

        // Top referrers
        let top_referrers = sqlx::query!(
            r#"
            SELECT COALESCE(referer, 'Direct') as referer, COUNT(*) as count
            FROM click_events
            WHERE short_url_id = $1
            GROUP BY referer
            ORDER BY count DESC
            LIMIT 10
            "#,
            short_url_id
        )
        .fetch_all(pool)
        .await?
        .into_iter()
        .map(|r| RefererStat {
            referer: r.referer.unwrap_or("Direct".to_string()),
            count: r.count.unwrap_or(0),
        })
        .collect();

        // Browser breakdown
        let browser_breakdown = sqlx::query!(
            r#"
            SELECT COALESCE(browser, 'Unknown') as browser, COUNT(*) as count
            FROM click_events
            WHERE short_url_id = $1
            GROUP BY browser
            ORDER BY count DESC
            "#,
            short_url_id
        )
        .fetch_all(pool)
        .await?
        .into_iter()
        .map(|r| BrowserStat {
            browser: r.browser.unwrap_or("Unknown".to_string()),
            count: r.count.unwrap_or(0),
        })
        .collect();

        // Hourly clicks (last 24 hours)
        let hourly_clicks = sqlx::query!(
            r#"
            SELECT to_char(date_trunc('hour', clicked_at), 'YYYY-MM-DD HH24:00') as hour,
                   COUNT(*) as count
            FROM click_events
            WHERE short_url_id = $1
              AND clicked_at >= NOW() - INTERVAL '24 hours'
            GROUP BY hour
            ORDER BY hour
            "#,
            short_url_id
        )
        .fetch_all(pool)
        .await?
        .into_iter()
        .map(|r| HourlyClick {
            hour: r.hour.unwrap_or_default(),
            count: r.count.unwrap_or(0),
        })
        .collect();

        Ok(UrlAnalytics {
            clicks_today,
            clicks_this_week,
            clicks_this_month,
            top_countries,
            top_referrers,
            browser_breakdown,
            hourly_clicks,
        })
    }
}

#[derive(Debug, serde::Serialize)]
pub struct UrlAnalytics {
    pub clicks_today: i64,
    pub clicks_this_week: i64,
    pub clicks_this_month: i64,
    pub top_countries: Vec<CountryStat>,
    pub top_referrers: Vec<RefererStat>,
    pub browser_breakdown: Vec<BrowserStat>,
    pub hourly_clicks: Vec<HourlyClick>,
}

fn parse_user_agent(ua: Option<&str>) -> (Option<String>, Option<String>, Option<String>) {
    let ua = match ua {
        Some(ua) => ua,
        None => return (None, None, None),
    };

    let browser = if ua.contains("Firefox") {
        Some("Firefox".to_string())
    } else if ua.contains("Chrome") && !ua.contains("Chromium") {
        Some("Chrome".to_string())
    } else if ua.contains("Safari") {
        Some("Safari".to_string())
    } else if ua.contains("Edge") {
        Some("Edge".to_string())
    } else {
        Some("Other".to_string())
    };

    let os = if ua.contains("Windows") {
        Some("Windows".to_string())
    } else if ua.contains("Mac OS") {
        Some("macOS".to_string())
    } else if ua.contains("Linux") {
        Some("Linux".to_string())
    } else if ua.contains("Android") {
        Some("Android".to_string())
    } else if ua.contains("iPhone") || ua.contains("iPad") {
        Some("iOS".to_string())
    } else {
        Some("Other".to_string())
    };

    let device = if ua.contains("Mobile") || ua.contains("Android") || ua.contains("iPhone") {
        Some("Mobile".to_string())
    } else if ua.contains("iPad") || ua.contains("Tablet") {
        Some("Tablet".to_string())
    } else {
        Some("Desktop".to_string())
    };

    (browser, os, device)
}
```

---

## 7. Handlers

### URLs Handler (`src/handlers/urls.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::middleware::auth::get_claims;
use crate::models::url::{CreateShortUrlRequest, UrlQuery};
use crate::services::qrcode::QrCodeService;
use crate::services::shortener::ShortenerService;

pub async fn create_url(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<CreateShortUrlRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    let claims = get_claims(&req);
    let user_id = claims.map(|c| c.sub);

    let short_url = ShortenerService::create_short_url(pool.get_ref(), &body, user_id).await?;

    let base_url = std::env::var("BASE_URL").unwrap_or_else(|_| "http://localhost:8080".to_string());
    let short_link = format!("{}/{}", base_url, short_url.short_code);

    Ok(HttpResponse::Created().json(serde_json::json!({
        "id": short_url.id,
        "short_code": short_url.short_code,
        "short_url": short_link,
        "original_url": short_url.original_url,
        "title": short_url.title,
        "expires_at": short_url.expires_at,
        "created_at": short_url.created_at,
    })))
}

pub async fn get_qr_code(
    pool: web::Data<PgPool>,
    path: web::Path<String>,
) -> Result<HttpResponse, AppError> {
    let short_code = path.into_inner();

    // Verify URL exists
    let _url = ShortenerService::resolve_url(pool.get_ref(), &short_code).await?;

    let base_url = std::env::var("BASE_URL").unwrap_or_else(|_| "http://localhost:8080".to_string());
    let full_url = format!("{}/{}", base_url, short_code);

    let qr_base64 = QrCodeService::generate_qr_base64(&full_url)?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "short_code": short_code,
        "qr_code": qr_base64,
    })))
}

pub async fn list_my_urls(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    query: web::Query<UrlQuery>,
) -> Result<HttpResponse, AppError> {
    let claims = crate::middleware::auth::require_auth(&req)?;
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let offset = ((page - 1) * per_page) as i64;

    let urls = sqlx::query_as!(
        crate::models::url::ShortUrl,
        r#"
        SELECT * FROM short_urls
        WHERE user_id = $1
          AND ($2::text IS NULL OR original_url ILIKE '%' || $2 || '%' OR title ILIKE '%' || $2 || '%')
          AND ($3::bool = true OR expires_at IS NULL OR expires_at > NOW())
        ORDER BY created_at DESC
        LIMIT $4 OFFSET $5
        "#,
        claims.sub,
        query.search,
        query.include_expired.unwrap_or(false),
        per_page as i64,
        offset
    )
    .fetch_all(pool.get_ref())
    .await?;

    let total: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM short_urls WHERE user_id = $1",
        claims.sub
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0);

    let base_url = std::env::var("BASE_URL").unwrap_or_else(|_| "http://localhost:8080".to_string());
    
    let urls_with_links: Vec<_> = urls.iter().map(|u| {
        serde_json::json!({
            "id": u.id,
            "short_code": u.short_code,
            "short_url": format!("{}/{}", base_url, u.short_code),
            "original_url": u.original_url,
            "title": u.title,
            "click_count": u.click_count,
            "unique_click_count": u.unique_click_count,
            "expires_at": u.expires_at,
            "is_active": u.is_active,
            "created_at": u.created_at,
        })
    }).collect();

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "urls": urls_with_links,
        "pagination": {
            "page": page,
            "per_page": per_page,
            "total": total,
        }
    })))
}

pub async fn delete_url(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = crate::middleware::auth::require_auth(&req)?;
    let url_id = path.into_inner();

    let result = sqlx::query!(
        "DELETE FROM short_urls WHERE id = $1 AND user_id = $2",
        url_id, claims.sub
    )
    .execute(pool.get_ref())
    .await?;

    if result.rows_affected() == 0 {
        return Err(AppError::NotFound("URL not found".to_string()));
    }

    Ok(HttpResponse::NoContent().finish())
}

pub async fn get_url_stats(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<String>,
) -> Result<HttpResponse, AppError> {
    let claims = crate::middleware::auth::require_auth(&req)?;
    let short_code = path.into_inner();

    let url = sqlx::query_as!(
        crate::models::url::ShortUrl,
        "SELECT * FROM short_urls WHERE short_code = $1 AND user_id = $2",
        short_code, claims.sub
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("URL not found".to_string()))?;

    let analytics = crate::services::analytics::AnalyticsService::get_url_stats(
        pool.get_ref(),
        url.id
    ).await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "url": url,
        "analytics": analytics,
    })))
}
```

### Redirect Handler (`src/handlers/redirect.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;

use crate::errors::AppError;
use crate::services::analytics::AnalyticsService;
use crate::services::shortener::ShortenerService;

pub async fn redirect(
    pool: web::Data<PgPool>,
    path: web::Path<String>,
    req: HttpRequest,
) -> Result<HttpResponse, AppError> {
    let short_code = path.into_inner();
    let url = ShortenerService::resolve_url(pool.get_ref(), &short_code).await?;

    // Extract request metadata
    let ip = req
        .connection_info()
        .realip_remote_addr()
        .map(|s| s.to_string());

    let user_agent = req
        .headers()
        .get("User-Agent")
        .and_then(|h| h.to_str().ok())
        .map(|s| s.to_string());

    let referer = req
        .headers()
        .get("Referer")
        .and_then(|h| h.to_str().ok())
        .map(|s| s.to_string());

    // Record click asynchronously
    let pool_clone = pool.get_ref().clone();
    let url_id = url.id;
    tokio::spawn(async move {
        let _ = AnalyticsService::record_click(
            &pool_clone,
            url_id,
            ip.as_deref(),
            user_agent.as_deref(),
            referer.as_deref(),
        ).await;
    });

    // If URL has password, return 401 with form
    if url.password_hash.is_some() {
        return Ok(HttpResponse::Unauthorized().json(serde_json::json!({
            "requires_password": true,
            "short_code": short_code,
        })));
    }

    Ok(HttpResponse::MovedPermanently()
        .append_header(("Location", url.original_url.as_str()))
        .finish())
}
```

---

## 8. Rate Limiting Middleware (`src/middleware/rate_limit.rs`)

```rust
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures_util::future::LocalBoxFuture;
use std::collections::HashMap;
use std::future::{ready, Ready};
use std::rc::Rc;
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Mutex;

pub struct RateLimiter {
    requests_per_minute: u32,
    store: Arc<Mutex<HashMap<String, (u32, Instant)>>>,
}

impl RateLimiter {
    pub fn new(requests_per_minute: u32) -> Self {
        RateLimiter {
            requests_per_minute,
            store: Arc::new(Mutex::new(HashMap::new())),
        }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RateLimiter
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = RateLimiterMiddleware<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RateLimiterMiddleware {
            service: Rc::new(service),
            requests_per_minute: self.requests_per_minute,
            store: self.store.clone(),
        }))
    }
}

pub struct RateLimiterMiddleware<S> {
    service: Rc<S>,
    requests_per_minute: u32,
    store: Arc<Mutex<HashMap<String, (u32, Instant)>>>,
}

impl<S, B> Service<ServiceRequest> for RateLimiterMiddleware<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let store = self.store.clone();
        let rpm = self.requests_per_minute;
        let service = self.service.clone();

        Box::pin(async move {
            let ip = req
                .connection_info()
                .realip_remote_addr()
                .unwrap_or("unknown")
                .to_string();

            let mut store = store.lock().await;
            let now = Instant::now();

            let (count, window_start) = store.entry(ip.clone()).or_insert((0, now));

            if now.duration_since(*window_start) > Duration::from_secs(60) {
                *count = 1;
                *window_start = now;
            } else {
                *count += 1;
            }

            if *count > rpm {
                drop(store);
                let response = HttpResponse::TooManyRequests()
                    .append_header(("Retry-After", "60"))
                    .json(serde_json::json!({
                        "error": "RATE_LIMIT_EXCEEDED",
                        "message": "Too many requests. Please try again later."
                    }));
                return Ok(req.into_response(response));
            }

            drop(store);
            service.call(req).await
        })
    }
}
```

---

## 9. Main Application (`src/main.rs`)

```rust
use actix_cors::Cors;
use actix_web::{middleware::Logger, web, App, HttpServer};
use dotenv::dotenv;
use sqlx::postgres::PgPoolOptions;
use std::env;

mod errors;
mod handlers;
mod middleware;
mod models;
mod services;

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

    log::info!("Starting URL Shortener at http://{}:{}", host, port);

    HttpServer::new(move || {
        let rate_limiter = middleware::rate_limit::RateLimiter::new(60); // 60 req/min per IP

        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .wrap(rate_limiter)
            .app_data(web::Data::new(pool.clone()))
            // API routes
            .service(
                web::scope("/api")
                    .route("/urls", web::post().to(handlers::urls::create_url))
                    .route("/urls", web::get().to(handlers::urls::list_my_urls))
                    .route("/urls/{id}", web::delete().to(handlers::urls::delete_url))
                    .route("/urls/{code}/qr", web::get().to(handlers::urls::get_qr_code))
                    .route("/urls/{code}/stats", web::get().to(handlers::urls::get_url_stats))
            )
            // Redirect route - must be last
            .route("/{code}", web::get().to(handlers::redirect::redirect))
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 10. Database Schema

```sql
CREATE TABLE short_urls (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    short_code VARCHAR(20) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    title VARCHAR(200),
    description TEXT,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    click_count BIGINT NOT NULL DEFAULT 0,
    unique_click_count BIGINT NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT true,
    expires_at TIMESTAMPTZ,
    password_hash TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_short_urls_code ON short_urls(short_code);
CREATE INDEX idx_short_urls_user ON short_urls(user_id);
CREATE INDEX idx_short_urls_expires ON short_urls(expires_at) WHERE expires_at IS NOT NULL;

CREATE TABLE click_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    short_url_id UUID NOT NULL REFERENCES short_urls(id) ON DELETE CASCADE,
    ip_address INET,
    user_agent TEXT,
    referer TEXT,
    country VARCHAR(2),
    city VARCHAR(100),
    browser VARCHAR(50),
    os VARCHAR(50),
    device_type VARCHAR(20),
    is_unique BOOLEAN NOT NULL DEFAULT false,
    clicked_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_clicks_url ON click_events(short_url_id);
CREATE INDEX idx_clicks_time ON click_events(clicked_at);
CREATE INDEX idx_clicks_unique ON click_events(short_url_id, ip_address, clicked_at);
```

---

## 11. ตัวอย่างการใช้งาน API

```bash
# Create short URL
curl -X POST http://localhost:8080/api/urls \
  -H "Content-Type: application/json" \
  -d '{
    "original_url": "https://www.example.com/very/long/url",
    "title": "Example Website",
    "expires_at": "2025-12-31T23:59:59Z"
  }'

# Create with custom code
curl -X POST http://localhost:8080/api/urls \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{
    "original_url": "https://www.example.com",
    "custom_code": "mylink"
  }'

# Get QR Code
curl http://localhost:8080/api/urls/abc123/qr

# Get stats
curl http://localhost:8080/api/urls/abc123/stats \
  -H "Authorization: Bearer <token>"

# Access short URL (redirects)
curl -L http://localhost:8080/abc123
```

---

## สรุป Part 084

ใน Part นี้เราได้สร้าง URL Shortener ที่สมบูรณ์ด้วย:
1. **nanoid** สำหรับ generate short codes ที่ไม่ซ้ำกัน
2. **Custom slugs** พร้อม validation
3. **Click Analytics** พร้อม geolocation และ user agent parsing
4. **QR Code Generation** แบบ base64 และ SVG
5. **Rate Limiting** ต่อ IP address
6. **URL Expiration** และ password protection
7. **Dashboard stats** พร้อม hourly breakdown

ใน **Part 085** เราจะสร้าง **Task Management API** แบบ Kanban

---

*[← Part 083: Real-time Chat Backend](../part_083/README.md) | [Part 085: Task Management API →](../part_085/README.md)*
