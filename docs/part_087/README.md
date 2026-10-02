# Part 087: Project: Analytics Dashboard API 📊

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง Event Tracking Ingestion System
- จัดเก็บ Time-series Data ใน PostgreSQL
- เขียน Aggregation Queries
- สร้าง Dashboard Metrics Endpoints
- ทำ User Funnel Analysis
- ทำ Cohort Analysis
- Export ข้อมูลเป็น CSV
- สร้าง Complete Analytics API

---

## 1. โครงสร้างโปรเจกต์

```
analytics_api/
├── Cargo.toml
├── .env
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── event.rs
    │   └── metrics.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── tracking.rs
    │   ├── dashboard.rs
    │   ├── funnels.rs
    │   ├── cohorts.rs
    │   └── export.rs
    └── services/
        ├── mod.rs
        ├── aggregation.rs
        └── export.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "analytics_api"
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
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
thiserror = "1"
csv = "1"
validator = { version = "0.18", features = ["derive"] }
flate2 = "1"
```

---

## 3. Models

### 3.1 Event Model (`src/models/event.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Event {
    pub id: Uuid,
    pub event_name: String,
    pub user_id: Option<String>,
    pub session_id: Option<String>,
    pub anonymous_id: Option<String>,
    pub project_id: String,
    pub properties: serde_json::Value,
    pub context: Option<serde_json::Value>,
    pub ip_address: Option<String>,
    pub user_agent: Option<String>,
    pub page_url: Option<String>,
    pub referrer: Option<String>,
    pub timestamp: DateTime<Utc>,
    pub received_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct TrackEventRequest {
    pub event: String,
    pub user_id: Option<String>,
    pub anonymous_id: Option<String>,
    pub session_id: Option<String>,
    pub properties: Option<serde_json::Value>,
    pub context: Option<serde_json::Value>,
    pub timestamp: Option<DateTime<Utc>>,
}

#[derive(Debug, Deserialize)]
pub struct BatchTrackRequest {
    pub batch: Vec<TrackEventRequest>,
}

#[derive(Debug, Deserialize)]
pub struct IdentifyRequest {
    pub user_id: String,
    pub anonymous_id: Option<String>,
    pub traits: serde_json::Value,
}

#[derive(Debug, Deserialize)]
pub struct PageViewRequest {
    pub user_id: Option<String>,
    pub anonymous_id: Option<String>,
    pub name: Option<String>,
    pub url: String,
    pub referrer: Option<String>,
    pub properties: Option<serde_json::Value>,
}
```

### 3.2 Metrics Model (`src/models/metrics.rs`)

```rust
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Deserialize)]
pub struct MetricsQuery {
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub granularity: Option<Granularity>,
    pub project_id: String,
    pub filters: Option<serde_json::Value>,
}

#[derive(Debug, Deserialize, Clone)]
#[serde(rename_all = "lowercase")]
pub enum Granularity {
    Hour,
    Day,
    Week,
    Month,
}

impl Granularity {
    pub fn to_sql(&self) -> &str {
        match self {
            Granularity::Hour => "hour",
            Granularity::Day => "day",
            Granularity::Week => "week",
            Granularity::Month => "month",
        }
    }
}

#[derive(Debug, Serialize)]
pub struct DashboardMetrics {
    pub total_events: i64,
    pub unique_users: i64,
    pub sessions: i64,
    pub page_views: i64,
    pub avg_session_duration_seconds: f64,
    pub bounce_rate: f64,
    pub events_over_time: Vec<TimeSeriesPoint>,
    pub top_events: Vec<EventCount>,
    pub top_pages: Vec<PageCount>,
    pub top_referrers: Vec<RefererCount>,
    pub device_breakdown: Vec<DeviceCount>,
    pub geo_breakdown: Vec<GeoCount>,
}

#[derive(Debug, Serialize)]
pub struct TimeSeriesPoint {
    pub timestamp: String,
    pub value: i64,
}

#[derive(Debug, Serialize)]
pub struct EventCount {
    pub event_name: String,
    pub count: i64,
    pub unique_users: i64,
}

#[derive(Debug, Serialize)]
pub struct PageCount {
    pub page_url: String,
    pub views: i64,
    pub unique_visitors: i64,
    pub avg_time_on_page: Option<f64>,
}

#[derive(Debug, Serialize)]
pub struct RefererCount {
    pub referrer: String,
    pub count: i64,
}

#[derive(Debug, Serialize)]
pub struct DeviceCount {
    pub device_type: String,
    pub count: i64,
    pub percentage: f64,
}

#[derive(Debug, Serialize)]
pub struct GeoCount {
    pub country: String,
    pub city: Option<String>,
    pub count: i64,
}

#[derive(Debug, Deserialize)]
pub struct FunnelQuery {
    pub project_id: String,
    pub steps: Vec<FunnelStep>,
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub conversion_window_hours: Option<i64>,
}

#[derive(Debug, Deserialize, Serialize)]
pub struct FunnelStep {
    pub event: String,
    pub filter: Option<serde_json::Value>,
}

#[derive(Debug, Serialize)]
pub struct FunnelResult {
    pub steps: Vec<FunnelStepResult>,
    pub overall_conversion_rate: f64,
}

#[derive(Debug, Serialize)]
pub struct FunnelStepResult {
    pub step: usize,
    pub event_name: String,
    pub users: i64,
    pub conversion_rate: f64,
    pub drop_off_rate: f64,
    pub avg_time_to_convert_seconds: Option<f64>,
}

#[derive(Debug, Deserialize)]
pub struct CohortQuery {
    pub project_id: String,
    pub cohort_event: String,
    pub return_event: String,
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub granularity: Option<Granularity>,
}
```

---

## 4. Event Tracking Handler (`src/handlers/tracking.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use chrono::Utc;
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;
use crate::models::event::{BatchTrackRequest, IdentifyRequest, PageViewRequest, TrackEventRequest};

fn get_project_id(req: &HttpRequest) -> String {
    req.headers()
        .get("X-Project-ID")
        .and_then(|h| h.to_str().ok())
        .unwrap_or("default")
        .to_string()
}

pub async fn track_event(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<TrackEventRequest>,
) -> Result<HttpResponse, AppError> {
    let project_id = get_project_id(&req);

    let ip = req.connection_info().realip_remote_addr()
        .map(|s| s.to_string());
    let user_agent = req.headers().get("User-Agent")
        .and_then(|h| h.to_str().ok())
        .map(|s| s.to_string());

    let event_id = Uuid::new_v4();
    let timestamp = body.timestamp.unwrap_or_else(Utc::now);
    let properties = body.properties.clone().unwrap_or(serde_json::json!({}));

    sqlx::query!(
        r#"
        INSERT INTO events (id, event_name, user_id, session_id, anonymous_id,
                           project_id, properties, context, ip_address, user_agent, timestamp)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
        "#,
        event_id,
        body.event,
        body.user_id,
        body.session_id,
        body.anonymous_id,
        project_id,
        properties,
        body.context,
        ip,
        user_agent,
        timestamp
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "event_id": event_id,
    })))
}

pub async fn track_batch(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<BatchTrackRequest>,
) -> Result<HttpResponse, AppError> {
    let project_id = get_project_id(&req);

    if body.batch.is_empty() || body.batch.len() > 1000 {
        return Err(AppError::BadRequest("Batch must have 1-1000 events".to_string()));
    }

    let ip = req.connection_info().realip_remote_addr()
        .map(|s| s.to_string());
    let user_agent = req.headers().get("User-Agent")
        .and_then(|h| h.to_str().ok())
        .map(|s| s.to_string());

    let mut tx = pool.begin().await?;

    for event_req in &body.batch {
        let event_id = Uuid::new_v4();
        let timestamp = event_req.timestamp.unwrap_or_else(Utc::now);
        let properties = event_req.properties.clone().unwrap_or(serde_json::json!({}));

        sqlx::query!(
            r#"
            INSERT INTO events (id, event_name, user_id, session_id, anonymous_id,
                               project_id, properties, context, ip_address, user_agent, timestamp)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
            "#,
            event_id,
            event_req.event,
            event_req.user_id,
            event_req.session_id,
            event_req.anonymous_id,
            project_id,
            properties,
            event_req.context,
            ip,
            user_agent,
            timestamp
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "events_tracked": body.batch.len(),
    })))
}

pub async fn track_page_view(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<PageViewRequest>,
) -> Result<HttpResponse, AppError> {
    let project_id = get_project_id(&req);
    let ip = req.connection_info().realip_remote_addr().map(|s| s.to_string());
    let user_agent = req.headers().get("User-Agent")
        .and_then(|h| h.to_str().ok())
        .map(|s| s.to_string());

    let properties = serde_json::json!({
        "url": body.url,
        "name": body.name,
        "referrer": body.referrer,
    });

    let event_id = Uuid::new_v4();
    sqlx::query!(
        r#"
        INSERT INTO events (id, event_name, user_id, anonymous_id,
                           project_id, properties, ip_address, user_agent,
                           page_url, referrer, timestamp)
        VALUES ($1, 'page_view', $2, $3, $4, $5, $6, $7, $8, $9, NOW())
        "#,
        event_id,
        body.user_id,
        body.anonymous_id,
        project_id,
        properties,
        ip,
        user_agent,
        body.url,
        body.referrer
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({ "success": true })))
}
```

---

## 5. Dashboard Handler (`src/handlers/dashboard.rs`)

```rust
use actix_web::{web, HttpResponse};
use sqlx::PgPool;

use crate::errors::AppError;
use crate::models::metrics::{DashboardMetrics, EventCount, GeoCount, MetricsQuery, TimeSeriesPoint};

pub async fn get_dashboard(
    pool: web::Data<PgPool>,
    query: web::Query<MetricsQuery>,
) -> Result<HttpResponse, AppError> {
    let granularity = query.granularity.clone().unwrap_or(crate::models::metrics::Granularity::Day);
    let gran_sql = granularity.to_sql();

    // Total events
    let total_events: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM events WHERE project_id = $1 AND timestamp BETWEEN $2 AND $3",
        query.project_id, query.start_date, query.end_date
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0);

    // Unique users
    let unique_users: i64 = sqlx::query_scalar!(
        r#"
        SELECT COUNT(DISTINCT COALESCE(user_id, anonymous_id))
        FROM events
        WHERE project_id = $1 AND timestamp BETWEEN $2 AND $3
        "#,
        query.project_id, query.start_date, query.end_date
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0);

    // Page views
    let page_views: i64 = sqlx::query_scalar!(
        "SELECT COUNT(*) FROM events WHERE project_id = $1 AND event_name = 'page_view' AND timestamp BETWEEN $2 AND $3",
        query.project_id, query.start_date, query.end_date
    )
    .fetch_one(pool.get_ref())
    .await?
    .unwrap_or(0);

    // Events over time
    let time_series = sqlx::query!(
        r#"
        SELECT date_trunc($4, timestamp) as bucket, COUNT(*) as count
        FROM events
        WHERE project_id = $1 AND timestamp BETWEEN $2 AND $3
        GROUP BY bucket
        ORDER BY bucket
        "#,
        query.project_id, query.start_date, query.end_date, gran_sql
    )
    .fetch_all(pool.get_ref())
    .await?
    .into_iter()
    .filter_map(|r| {
        r.bucket.map(|b| TimeSeriesPoint {
            timestamp: b.to_string(),
            count: r.count.unwrap_or(0),
        })
    })
    .collect::<Vec<_>>();

    // Top events
    let top_events = sqlx::query!(
        r#"
        SELECT event_name,
               COUNT(*) as count,
               COUNT(DISTINCT COALESCE(user_id, anonymous_id)) as unique_users
        FROM events
        WHERE project_id = $1 AND timestamp BETWEEN $2 AND $3
        GROUP BY event_name
        ORDER BY count DESC
        LIMIT 20
        "#,
        query.project_id, query.start_date, query.end_date
    )
    .fetch_all(pool.get_ref())
    .await?
    .into_iter()
    .map(|r| EventCount {
        event_name: r.event_name,
        count: r.count.unwrap_or(0),
        unique_users: r.unique_users.unwrap_or(0),
    })
    .collect();

    // Top pages
    let top_pages = sqlx::query!(
        r#"
        SELECT page_url, COUNT(*) as views,
               COUNT(DISTINCT COALESCE(user_id, anonymous_id)) as unique_visitors
        FROM events
        WHERE project_id = $1 AND event_name = 'page_view'
          AND timestamp BETWEEN $2 AND $3
          AND page_url IS NOT NULL
        GROUP BY page_url
        ORDER BY views DESC
        LIMIT 20
        "#,
        query.project_id, query.start_date, query.end_date
    )
    .fetch_all(pool.get_ref())
    .await?
    .into_iter()
    .map(|r| crate::models::metrics::PageCount {
        page_url: r.page_url.unwrap_or_default(),
        views: r.views.unwrap_or(0),
        unique_visitors: r.unique_visitors.unwrap_or(0),
        avg_time_on_page: None,
    })
    .collect();

    Ok(HttpResponse::Ok().json(DashboardMetrics {
        total_events,
        unique_users,
        sessions: 0, // Would need session tracking logic
        page_views,
        avg_session_duration_seconds: 0.0,
        bounce_rate: 0.0,
        events_over_time: time_series.into_iter().map(|t| TimeSeriesPoint {
            timestamp: t.timestamp,
            value: t.count,
        }).collect(),
        top_events,
        top_pages,
        top_referrers: vec![],
        device_breakdown: vec![],
        geo_breakdown: vec![],
    }))
}

// Helper struct for time series mapping
struct TimeSeriesPoint {
    timestamp: String,
    count: i64,
}
```

---

## 6. Funnel Analysis Handler (`src/handlers/funnels.rs`)

```rust
use actix_web::{web, HttpResponse};
use sqlx::PgPool;

use crate::errors::AppError;
use crate::models::metrics::{FunnelQuery, FunnelResult, FunnelStepResult};

pub async fn analyze_funnel(
    pool: web::Data<PgPool>,
    body: web::Json<FunnelQuery>,
) -> Result<HttpResponse, AppError> {
    if body.steps.is_empty() || body.steps.len() > 10 {
        return Err(AppError::BadRequest("Funnel must have 1-10 steps".to_string()));
    }

    let conversion_window = body.conversion_window_hours.unwrap_or(24);
    let mut step_results: Vec<FunnelStepResult> = Vec::new();
    let mut previous_users: i64 = 0;

    // Calculate users at each step
    for (idx, step) in body.steps.iter().enumerate() {
        let user_count = if idx == 0 {
            // First step: count users who performed this event
            sqlx::query_scalar!(
                r#"
                SELECT COUNT(DISTINCT COALESCE(user_id, anonymous_id))
                FROM events
                WHERE project_id = $1
                  AND event_name = $2
                  AND timestamp BETWEEN $3 AND $4
                "#,
                body.project_id,
                step.event,
                body.start_date,
                body.end_date
            )
            .fetch_one(pool.get_ref())
            .await?
            .unwrap_or(0)
        } else {
            // Subsequent steps: count users who also performed the previous step
            let prev_step = &body.steps[idx - 1];
            sqlx::query_scalar!(
                r#"
                SELECT COUNT(DISTINCT e2.user_id)
                FROM events e1
                JOIN events e2 ON COALESCE(e1.user_id, e1.anonymous_id) = COALESCE(e2.user_id, e2.anonymous_id)
                WHERE e1.project_id = $1
                  AND e1.event_name = $2
                  AND e1.timestamp BETWEEN $3 AND $4
                  AND e2.event_name = $5
                  AND e2.timestamp > e1.timestamp
                  AND e2.timestamp <= e1.timestamp + INTERVAL '1 hour' * $6
                "#,
                body.project_id,
                prev_step.event,
                body.start_date,
                body.end_date,
                step.event,
                conversion_window as f64
            )
            .fetch_one(pool.get_ref())
            .await?
            .unwrap_or(0)
        };

        let conversion_rate = if idx == 0 || previous_users == 0 {
            100.0
        } else {
            (user_count as f64 / previous_users as f64) * 100.0
        };

        let drop_off_rate = 100.0 - conversion_rate;

        step_results.push(FunnelStepResult {
            step: idx + 1,
            event_name: step.event.clone(),
            users: user_count,
            conversion_rate,
            drop_off_rate,
            avg_time_to_convert_seconds: None,
        });

        previous_users = user_count;
    }

    let first_step = step_results.first().map(|s| s.users).unwrap_or(1);
    let last_step = step_results.last().map(|s| s.users).unwrap_or(0);
    let overall_conversion = if first_step > 0 {
        (last_step as f64 / first_step as f64) * 100.0
    } else {
        0.0
    };

    Ok(HttpResponse::Ok().json(FunnelResult {
        steps: step_results,
        overall_conversion_rate: overall_conversion,
    }))
}
```

---

## 7. Cohort Analysis Handler (`src/handlers/cohorts.rs`)

```rust
use actix_web::{web, HttpResponse};
use sqlx::PgPool;

use crate::errors::AppError;
use crate::models::metrics::CohortQuery;

pub async fn analyze_cohort(
    pool: web::Data<PgPool>,
    body: web::Json<CohortQuery>,
) -> Result<HttpResponse, AppError> {
    let granularity = body.granularity.clone()
        .unwrap_or(crate::models::metrics::Granularity::Week);
    let gran_sql = granularity.to_sql();

    // Get cohorts: users grouped by when they first performed the cohort event
    let cohorts = sqlx::query!(
        r#"
        SELECT date_trunc($4, first_seen) as cohort_period,
               COUNT(*) as cohort_size,
               array_agg(DISTINCT user_id) as user_ids
        FROM (
            SELECT COALESCE(user_id, anonymous_id) as user_id,
                   MIN(timestamp) as first_seen
            FROM events
            WHERE project_id = $1
              AND event_name = $2
              AND timestamp BETWEEN $3 AND NOW()
            GROUP BY user_id
        ) cohort_users
        GROUP BY cohort_period
        ORDER BY cohort_period
        LIMIT 12
        "#,
        body.project_id,
        body.cohort_event,
        body.start_date,
        gran_sql
    )
    .fetch_all(pool.get_ref())
    .await?;

    // For each cohort, calculate retention
    let mut cohort_data = Vec::new();

    for cohort in &cohorts {
        let cohort_period = cohort.cohort_period
            .map(|t| t.to_string())
            .unwrap_or_default();
        let cohort_size = cohort.cohort_size.unwrap_or(0);

        // Calculate retention for each subsequent period
        let retention = sqlx::query!(
            r#"
            SELECT period_num, COUNT(DISTINCT returning_user) as returning_count
            FROM (
                SELECT
                    COALESCE(e.user_id, e.anonymous_id) as returning_user,
                    EXTRACT(EPOCH FROM (date_trunc($6, e.timestamp) - $3::timestamptz))
                        / EXTRACT(EPOCH FROM INTERVAL '1 ' || $6) as period_num
                FROM events e
                WHERE e.project_id = $1
                  AND e.event_name = $2
                  AND COALESCE(e.user_id, e.anonymous_id) = ANY($4::text[])
                  AND e.timestamp >= $3
                  AND e.timestamp <= $5
            ) retention_data
            WHERE period_num >= 0 AND period_num <= 12
            GROUP BY period_num
            ORDER BY period_num
            "#,
            body.project_id,
            body.return_event,
            cohort.cohort_period,
            cohort.user_ids.as_ref().unwrap_or(&vec![]) as &Vec<Option<String>>,
            body.end_date,
            gran_sql
        )
        .fetch_all(pool.get_ref())
        .await?;

        cohort_data.push(serde_json::json!({
            "cohort_period": cohort_period,
            "cohort_size": cohort_size,
            "retention": retention.iter().map(|r| {
                serde_json::json!({
                    "period": r.period_num,
                    "users": r.returning_count,
                    "rate": if cohort_size > 0 {
                        (r.returning_count.unwrap_or(0) as f64 / cohort_size as f64) * 100.0
                    } else { 0.0 }
                })
            }).collect::<Vec<_>>()
        }));
    }

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "granularity": gran_sql,
        "cohorts": cohort_data,
    })))
}
```

---

## 8. Export Handler (`src/handlers/export.rs`)

```rust
use actix_web::{web, HttpResponse};
use chrono::{DateTime, Utc};
use csv::WriterBuilder;
use serde::Deserialize;
use sqlx::PgPool;

use crate::errors::AppError;

#[derive(Debug, Deserialize)]
pub struct ExportQuery {
    pub project_id: String,
    pub start_date: DateTime<Utc>,
    pub end_date: DateTime<Utc>,
    pub event_names: Option<Vec<String>>,
    pub limit: Option<i64>,
}

pub async fn export_events_csv(
    pool: web::Data<PgPool>,
    query: web::Query<ExportQuery>,
) -> Result<HttpResponse, AppError> {
    let limit = query.limit.unwrap_or(100_000).min(1_000_000);

    let events = sqlx::query!(
        r#"
        SELECT id, event_name, user_id, anonymous_id, session_id,
               properties::text, ip_address, user_agent, page_url,
               referrer, timestamp
        FROM events
        WHERE project_id = $1
          AND timestamp BETWEEN $2 AND $3
          AND ($4::text[] IS NULL OR event_name = ANY($4))
        ORDER BY timestamp DESC
        LIMIT $5
        "#,
        query.project_id,
        query.start_date,
        query.end_date,
        query.event_names.as_deref(),
        limit
    )
    .fetch_all(pool.get_ref())
    .await?;

    // Build CSV
    let mut wtr = WriterBuilder::new()
        .has_headers(true)
        .from_writer(vec![]);

    wtr.write_record(&[
        "id", "event_name", "user_id", "anonymous_id", "session_id",
        "properties", "ip_address", "user_agent", "page_url", "referrer", "timestamp"
    ]).map_err(|e| AppError::InternalError(e.to_string()))?;

    for event in &events {
        wtr.write_record(&[
            event.id.to_string(),
            event.event_name.clone(),
            event.user_id.clone().unwrap_or_default(),
            event.anonymous_id.clone().unwrap_or_default(),
            event.session_id.clone().unwrap_or_default(),
            event.properties.clone().unwrap_or_default(),
            event.ip_address.clone().unwrap_or_default(),
            event.user_agent.clone().unwrap_or_default(),
            event.page_url.clone().unwrap_or_default(),
            event.referrer.clone().unwrap_or_default(),
            event.timestamp.to_rfc3339(),
        ]).map_err(|e| AppError::InternalError(e.to_string()))?;
    }

    let csv_data = wtr.into_inner()
        .map_err(|e| AppError::InternalError(e.to_string()))?;

    let filename = format!(
        "events_{}_{}.csv",
        query.project_id,
        chrono::Utc::now().format("%Y%m%d_%H%M%S")
    );

    Ok(HttpResponse::Ok()
        .content_type("text/csv")
        .append_header(("Content-Disposition", format!("attachment; filename=\"{}\"", filename)))
        .body(csv_data))
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
        .max_connections(20)
        .connect(&database_url)
        .await
        .expect("Failed to create pool");

    log::info!("Starting Analytics API at http://{}:{}", host, port);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .app_data(web::Data::new(pool.clone()))
            .service(
                web::scope("/api")
                    // Event Ingestion
                    .route("/track", web::post().to(handlers::tracking::track_event))
                    .route("/track/batch", web::post().to(handlers::tracking::track_batch))
                    .route("/page", web::post().to(handlers::tracking::track_page_view))
                    // Dashboard
                    .route("/dashboard", web::get().to(handlers::dashboard::get_dashboard))
                    .route("/dashboard/realtime", web::get().to(handlers::dashboard::get_realtime))
                    // Funnels
                    .route("/funnels", web::post().to(handlers::funnels::analyze_funnel))
                    // Cohorts
                    .route("/cohorts", web::post().to(handlers::cohorts::analyze_cohort))
                    // Export
                    .route("/export/events", web::get().to(handlers::export::export_events_csv))
            )
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 10. Database Schema

```sql
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_name VARCHAR(100) NOT NULL,
    user_id VARCHAR(255),
    session_id VARCHAR(255),
    anonymous_id VARCHAR(255),
    project_id VARCHAR(100) NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',
    context JSONB,
    ip_address INET,
    user_agent TEXT,
    page_url TEXT,
    referrer TEXT,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    received_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (timestamp);

-- Create monthly partitions
CREATE TABLE events_2025_01 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
CREATE TABLE events_2025_02 PARTITION OF events
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

CREATE INDEX idx_events_project_time ON events(project_id, timestamp DESC);
CREATE INDEX idx_events_name ON events(event_name, project_id);
CREATE INDEX idx_events_user ON events(user_id, project_id) WHERE user_id IS NOT NULL;
CREATE INDEX idx_events_props ON events USING GIN(properties);

-- Pre-aggregated metrics table for faster dashboard queries
CREATE TABLE hourly_metrics (
    project_id VARCHAR(100) NOT NULL,
    hour TIMESTAMPTZ NOT NULL,
    event_name VARCHAR(100) NOT NULL,
    total_events INTEGER NOT NULL DEFAULT 0,
    unique_users INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (project_id, hour, event_name)
);
```

---

## 11. ตัวอย่างการใช้งาน

```bash
# Track an event
curl -X POST http://localhost:8080/api/track \
  -H "Content-Type: application/json" \
  -H "X-Project-ID: my-project" \
  -d '{"event": "button_clicked", "user_id": "user123", "properties": {"button": "signup"}}'

# Get dashboard metrics
curl "http://localhost:8080/api/dashboard?project_id=my-project&start_date=2025-01-01T00:00:00Z&end_date=2025-01-31T23:59:59Z&granularity=day"

# Analyze funnel
curl -X POST http://localhost:8080/api/funnels \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "my-project",
    "start_date": "2025-01-01T00:00:00Z",
    "end_date": "2025-01-31T23:59:59Z",
    "steps": [
      {"event": "page_view"},
      {"event": "signup_started"},
      {"event": "signup_completed"}
    ],
    "conversion_window_hours": 24
  }'

# Export events as CSV
curl "http://localhost:8080/api/export/events?project_id=my-project&start_date=2025-01-01T00:00:00Z&end_date=2025-01-31T23:59:59Z" \
  -o events.csv
```

---

## สรุป Part 087

ใน Part นี้เราได้สร้าง Analytics Dashboard API ที่สมบูรณ์ด้วย:
1. **Event Ingestion** รองรับ single event, batch, และ page views
2. **Time-series Queries** พร้อม partitioned tables
3. **Dashboard Metrics** พร้อม multiple dimensions
4. **Funnel Analysis** ด้วย multi-step conversion tracking
5. **Cohort Analysis** พร้อม retention rates
6. **CSV Export** สำหรับ raw event data

ใน **Part 088** เราจะสร้าง **File Storage Service** แบบ S3-compatible

---

*[← Part 086: Notification Service](../part_086/README.md) | [Part 088: File Storage Service →](../part_088/README.md)*
