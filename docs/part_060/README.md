# Part 060: Webhooks 🪝

## 🎯 เป้าหมายของ Part นี้

- Webhook design patterns
- Signature verification (HMAC-SHA256)
- Retry ด้วย exponential backoff
- Webhook delivery tracking
- Webhook management API (create/delete/list)
- Event filtering
- Idempotency
- สร้าง Webhook System

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"
reqwest = { version = "0.11", features = ["json"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio-native-tls", "chrono", "uuid"] }
thiserror = "1"
anyhow = "1"
tokio-retry = "0.3"
```

---

## 2. Webhook Models

```rust
// src/webhook/models.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

// Webhook subscription
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Webhook {
    pub id: Uuid,
    pub name: String,
    pub url: String,
    pub secret: String,
    pub events: Vec<String>,      // Event types ที่ subscribe
    pub active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub last_triggered_at: Option<DateTime<Utc>>,
    pub metadata: serde_json::Value,
}

impl Webhook {
    pub fn new(name: &str, url: &str, events: Vec<String>) -> Self {
        Self {
            id: Uuid::new_v4(),
            name: name.to_string(),
            url: url.to_string(),
            secret: generate_webhook_secret(),
            events,
            active: true,
            created_at: Utc::now(),
            updated_at: Utc::now(),
            last_triggered_at: None,
            metadata: serde_json::Value::Null,
        }
    }

    pub fn subscribes_to(&self, event_type: &str) -> bool {
        self.events.iter().any(|e| {
            e == "*" || e == event_type || event_matches_pattern(e, event_type)
        })
    }
}

fn generate_webhook_secret() -> String {
    use sha2::{Sha256, Digest};
    let nonce = Uuid::new_v4().to_string();
    hex::encode(Sha256::digest(nonce.as_bytes()))
}

fn event_matches_pattern(pattern: &str, event: &str) -> bool {
    if let Some(prefix) = pattern.strip_suffix("*") {
        event.starts_with(prefix)
    } else {
        pattern == event
    }
}

// Webhook delivery record
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WebhookDelivery {
    pub id: Uuid,
    pub webhook_id: Uuid,
    pub event_id: String,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub status: DeliveryStatus,
    pub attempts: u32,
    pub response_status: Option<u16>,
    pub response_body: Option<String>,
    pub error: Option<String>,
    pub created_at: DateTime<Utc>,
    pub delivered_at: Option<DateTime<Utc>>,
    pub next_retry_at: Option<DateTime<Utc>>,
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum DeliveryStatus {
    Pending,
    Delivering,
    Delivered,
    Failed,
    Abandoned,  // หมดจำนวน retry
}

// Webhook event
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct WebhookEvent {
    pub id: String,          // Event ID สำหรับ idempotency
    pub event_type: String,
    pub payload: serde_json::Value,
    pub created_at: DateTime<Utc>,
    pub api_version: String,
}

impl WebhookEvent {
    pub fn new(event_type: &str, payload: serde_json::Value) -> Self {
        Self {
            id: format!("evt_{}", Uuid::new_v4().to_string().replace('-', "")),
            event_type: event_type.to_string(),
            payload,
            created_at: Utc::now(),
            api_version: "2024-01-01".to_string(),
        }
    }
}
```

---

## 3. Signature Verification

```rust
// src/webhook/signature.rs
use hmac::{Hmac, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

/// สร้าง signature สำหรับ payload
pub fn generate_signature(payload: &[u8], secret: &str) -> String {
    let mut mac = HmacSha256::new_from_slice(secret.as_bytes())
        .expect("HMAC can take key of any size");
    mac.update(payload);
    let result = mac.finalize().into_bytes();
    format!("sha256={}", hex::encode(result))
}

/// ตรวจสอบ signature
pub fn verify_signature(
    payload: &[u8],
    secret: &str,
    received_sig: &str,
) -> bool {
    let expected = generate_signature(payload, secret);

    // Constant time comparison ป้องกัน timing attacks
    constant_time_eq(expected.as_bytes(), received_sig.as_bytes())
}

fn constant_time_eq(a: &[u8], b: &[u8]) -> bool {
    if a.len() != b.len() {
        return false;
    }

    let mut result = 0u8;
    for (x, y) in a.iter().zip(b.iter()) {
        result |= x ^ y;
    }
    result == 0
}

/// สร้าง headers สำหรับ webhook delivery
pub fn build_webhook_headers(
    event: &super::models::WebhookEvent,
    payload: &[u8],
    secret: &str,
) -> Vec<(String, String)> {
    let signature = generate_signature(payload, secret);
    let timestamp = event.created_at.timestamp();

    vec![
        ("Content-Type".to_string(), "application/json".to_string()),
        ("X-Webhook-Signature".to_string(), signature),
        ("X-Webhook-Timestamp".to_string(), timestamp.to_string()),
        ("X-Webhook-Event".to_string(), event.event_type.clone()),
        ("X-Webhook-ID".to_string(), event.id.clone()),
    ]
}

// ตัวอย่าง: ตรวจสอบ webhook ที่รับเข้ามา
pub fn verify_incoming_webhook(
    req_body: &[u8],
    signature_header: &str,
    secret: &str,
    timestamp_header: &str,
) -> Result<(), WebhookVerifyError> {
    // ตรวจสอบ timestamp (ป้องกัน replay)
    let timestamp: i64 = timestamp_header
        .parse()
        .map_err(|_| WebhookVerifyError::InvalidTimestamp)?;

    let now = chrono::Utc::now().timestamp();
    if (now - timestamp).abs() > 300 {
        return Err(WebhookVerifyError::ExpiredTimestamp);
    }

    // ตรวจสอบ signature
    if !verify_signature(req_body, secret, signature_header) {
        return Err(WebhookVerifyError::InvalidSignature);
    }

    Ok(())
}

#[derive(Debug, thiserror::Error)]
pub enum WebhookVerifyError {
    #[error("Invalid timestamp format")]
    InvalidTimestamp,
    #[error("Timestamp expired")]
    ExpiredTimestamp,
    #[error("Invalid signature")]
    InvalidSignature,
}
```

---

## 4. Webhook Delivery with Retry

```rust
// src/webhook/delivery.rs
use reqwest::Client;
use std::time::Duration;
use crate::webhook::models::{Webhook, WebhookDelivery, WebhookEvent, DeliveryStatus};
use crate::webhook::signature::build_webhook_headers;

pub struct WebhookDeliverer {
    http_client: Client,
    max_retries: u32,
}

impl WebhookDeliverer {
    pub fn new(max_retries: u32) -> Self {
        let http_client = Client::builder()
            .timeout(Duration::from_secs(30))
            .build()
            .expect("Failed to build HTTP client");

        Self {
            http_client,
            max_retries,
        }
    }

    // ส่ง webhook พร้อม retry
    pub async fn deliver(
        &self,
        webhook: &Webhook,
        event: &WebhookEvent,
    ) -> WebhookDelivery {
        let payload = serde_json::to_vec(event).unwrap_or_default();
        let headers = build_webhook_headers(event, &payload, &webhook.secret);

        let mut delivery = WebhookDelivery {
            id: uuid::Uuid::new_v4(),
            webhook_id: webhook.id,
            event_id: event.id.clone(),
            event_type: event.event_type.clone(),
            payload: event.payload.clone(),
            status: DeliveryStatus::Delivering,
            attempts: 0,
            response_status: None,
            response_body: None,
            error: None,
            created_at: chrono::Utc::now(),
            delivered_at: None,
            next_retry_at: None,
        };

        for attempt in 1..=self.max_retries {
            delivery.attempts = attempt;

            match self.send_once(&webhook.url, &payload, &headers).await {
                Ok((status, body)) => {
                    delivery.response_status = Some(status);
                    delivery.response_body = Some(body);

                    if status >= 200 && status < 300 {
                        delivery.status = DeliveryStatus::Delivered;
                        delivery.delivered_at = Some(chrono::Utc::now());
                        return delivery;
                    } else {
                        delivery.error = Some(format!("HTTP {}", status));
                    }
                }
                Err(e) => {
                    delivery.error = Some(e.to_string());
                }
            }

            if attempt < self.max_retries {
                let delay = self.backoff_delay(attempt);
                delivery.next_retry_at =
                    Some(chrono::Utc::now() + chrono::Duration::milliseconds(delay as i64));

                println!(
                    "[Webhook] Retry {}/{} in {}ms for webhook {}",
                    attempt, self.max_retries, delay, webhook.id
                );

                tokio::time::sleep(Duration::from_millis(delay)).await;
            }
        }

        delivery.status = DeliveryStatus::Abandoned;
        delivery
    }

    async fn send_once(
        &self,
        url: &str,
        payload: &[u8],
        headers: &[(String, String)],
    ) -> anyhow::Result<(u16, String)> {
        let mut req = self.http_client.post(url).body(payload.to_vec());

        for (key, value) in headers {
            req = req.header(key.as_str(), value.as_str());
        }

        let response = req.send().await?;
        let status = response.status().as_u16();
        let body = response.text().await.unwrap_or_default();

        Ok((status, body))
    }

    // Exponential backoff: 1s, 2s, 4s, 8s, 16s...
    fn backoff_delay(&self, attempt: u32) -> u64 {
        let base = 1000u64; // 1 second
        let max = 3600_000u64; // 1 hour max

        let delay = base * 2u64.pow(attempt - 1);
        delay.min(max)
    }
}
```

---

## 5. Webhook Manager

```rust
// src/webhook/manager.rs
use std::sync::Arc;
use tokio::sync::RwLock;
use crate::webhook::{
    models::{Webhook, WebhookDelivery, WebhookEvent},
    delivery::WebhookDeliverer,
};

pub struct WebhookManager {
    webhooks: Arc<RwLock<Vec<Webhook>>>,
    deliveries: Arc<RwLock<Vec<WebhookDelivery>>>,
    deliverer: Arc<WebhookDeliverer>,
}

impl WebhookManager {
    pub fn new() -> Self {
        Self {
            webhooks: Arc::new(RwLock::new(Vec::new())),
            deliveries: Arc::new(RwLock::new(Vec::new())),
            deliverer: Arc::new(WebhookDeliverer::new(5)),
        }
    }

    // Register webhook
    pub async fn register(&self, webhook: Webhook) -> Webhook {
        let w = webhook.clone();
        self.webhooks.write().await.push(webhook);
        println!("[WebhookManager] Registered: {} -> {}", w.name, w.url);
        w
    }

    // Trigger event - ส่ง webhook ไปยัง subscribers ทั้งหมด
    pub async fn trigger(&self, event: WebhookEvent) {
        let webhooks = self.webhooks.read().await.clone();
        let event = Arc::new(event);

        let mut tasks = Vec::new();

        for webhook in webhooks {
            if !webhook.active || !webhook.subscribes_to(&event.event_type) {
                continue;
            }

            let deliverer = self.deliverer.clone();
            let deliveries = self.deliveries.clone();
            let event = event.clone();
            let webhook = webhook.clone();

            tasks.push(tokio::spawn(async move {
                let delivery = deliverer.deliver(&webhook, &event).await;

                println!(
                    "[WebhookManager] Delivered to {} - status: {:?} (attempt {})",
                    webhook.url, delivery.status, delivery.attempts
                );

                deliveries.write().await.push(delivery);
            }));
        }

        // รอทุก deliveries
        for task in tasks {
            let _ = task.await;
        }
    }

    // Get webhook by ID
    pub async fn get(&self, id: &uuid::Uuid) -> Option<Webhook> {
        self.webhooks.read().await.iter().find(|w| &w.id == id).cloned()
    }

    // List all webhooks
    pub async fn list(&self) -> Vec<Webhook> {
        self.webhooks.read().await.clone()
    }

    // Update webhook
    pub async fn update(
        &self,
        id: &uuid::Uuid,
        updates: WebhookUpdate,
    ) -> Option<Webhook> {
        let mut webhooks = self.webhooks.write().await;

        if let Some(webhook) = webhooks.iter_mut().find(|w| &w.id == id) {
            if let Some(url) = updates.url {
                webhook.url = url;
            }
            if let Some(events) = updates.events {
                webhook.events = events;
            }
            if let Some(active) = updates.active {
                webhook.active = active;
            }
            webhook.updated_at = chrono::Utc::now();

            Some(webhook.clone())
        } else {
            None
        }
    }

    // Delete webhook
    pub async fn delete(&self, id: &uuid::Uuid) -> bool {
        let mut webhooks = self.webhooks.write().await;
        let before = webhooks.len();
        webhooks.retain(|w| &w.id != id);
        webhooks.len() < before
    }

    // Get deliveries for a webhook
    pub async fn get_deliveries(&self, webhook_id: &uuid::Uuid) -> Vec<WebhookDelivery> {
        self.deliveries
            .read()
            .await
            .iter()
            .filter(|d| &d.webhook_id == webhook_id)
            .cloned()
            .collect()
    }
}

#[derive(Debug, serde::Deserialize)]
pub struct WebhookUpdate {
    pub url: Option<String>,
    pub events: Option<Vec<String>>,
    pub active: Option<bool>,
}
```

---

## 6. Receiving Webhooks (Incoming)

```rust
// src/webhook/receiver.rs
use actix_web::{web, HttpRequest, HttpResponse};
use crate::webhook::signature::verify_incoming_webhook;

// Handler สำหรับรับ webhook จาก external service
pub async fn receive_webhook(
    req: HttpRequest,
    body: web::Bytes,
    config: web::Data<WebhookReceiverConfig>,
) -> HttpResponse {
    // อ่าน headers
    let signature = req
        .headers()
        .get("X-Webhook-Signature")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("");

    let timestamp = req
        .headers()
        .get("X-Webhook-Timestamp")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("0");

    let event_type = req
        .headers()
        .get("X-Webhook-Event")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("unknown");

    let event_id = req
        .headers()
        .get("X-Webhook-ID")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("");

    // ตรวจสอบ signature
    if let Err(e) = verify_incoming_webhook(&body, signature, &config.secret, timestamp) {
        return HttpResponse::Unauthorized().json(serde_json::json!({
            "error": format!("Webhook verification failed: {}", e)
        }));
    }

    // Idempotency check - ตรวจว่าเคยประมวลผล event นี้แล้วหรือยัง
    if !event_id.is_empty() {
        if config.processed_events.contains(event_id) {
            return HttpResponse::Ok().json(serde_json::json!({
                "status": "already_processed",
                "event_id": event_id
            }));
        }
        config.processed_events.mark_processed(event_id);
    }

    // Parse payload
    let payload: serde_json::Value = match serde_json::from_slice(&body) {
        Ok(p) => p,
        Err(e) => {
            return HttpResponse::BadRequest().json(serde_json::json!({
                "error": format!("Invalid JSON: {}", e)
            }))
        }
    };

    // Handle event
    println!("[Receiver] Event: {} - ID: {}", event_type, event_id);

    match event_type {
        "order.created" => handle_order_created(&payload).await,
        "payment.completed" => handle_payment_completed(&payload).await,
        _ => println!("[Receiver] Unhandled event: {}", event_type),
    }

    HttpResponse::Ok().json(serde_json::json!({ "received": true }))
}

async fn handle_order_created(payload: &serde_json::Value) {
    println!("[Receiver] Order created: {}", payload["order_id"]);
}

async fn handle_payment_completed(payload: &serde_json::Value) {
    println!("[Receiver] Payment completed: {}", payload["payment_id"]);
}

pub struct WebhookReceiverConfig {
    pub secret: String,
    pub processed_events: ProcessedEventTracker,
}

pub struct ProcessedEventTracker {
    events: std::sync::Arc<tokio::sync::RwLock<std::collections::HashSet<String>>>,
}

impl ProcessedEventTracker {
    pub fn new() -> Self {
        Self {
            events: std::sync::Arc::new(tokio::sync::RwLock::new(
                std::collections::HashSet::new(),
            )),
        }
    }

    pub fn contains(&self, event_id: &str) -> bool {
        // Synchronous check ด้วย try_read
        self.events
            .try_read()
            .map(|e| e.contains(event_id))
            .unwrap_or(false)
    }

    pub fn mark_processed(&self, event_id: &str) {
        let events = self.events.clone();
        let id = event_id.to_string();
        tokio::spawn(async move {
            events.write().await.insert(id);
        });
    }
}
```

---

## 7. Webhook Management API

```rust
// src/webhook/api.rs
use actix_web::{web, HttpResponse};
use serde::Deserialize;
use uuid::Uuid;
use crate::webhook::{
    manager::{WebhookManager, WebhookUpdate},
    models::{Webhook, WebhookEvent},
};

// Create webhook
#[derive(Deserialize)]
pub struct CreateWebhookRequest {
    pub name: String,
    pub url: String,
    pub events: Vec<String>,
}

pub async fn create_webhook(
    manager: web::Data<WebhookManager>,
    body: web::Json<CreateWebhookRequest>,
) -> HttpResponse {
    // Validate URL
    if !body.url.starts_with("https://") && !body.url.starts_with("http://") {
        return HttpResponse::BadRequest().json(serde_json::json!({
            "error": "URL must start with http:// or https://"
        }));
    }

    let webhook = Webhook::new(&body.name, &body.url, body.events.clone());
    let created = manager.register(webhook).await;

    HttpResponse::Created().json(created)
}

// List webhooks
pub async fn list_webhooks(
    manager: web::Data<WebhookManager>,
) -> HttpResponse {
    let webhooks = manager.list().await;
    HttpResponse::Ok().json(webhooks)
}

// Get webhook
pub async fn get_webhook(
    manager: web::Data<WebhookManager>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();

    match manager.get(&id).await {
        Some(webhook) => HttpResponse::Ok().json(webhook),
        None => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Webhook not found"
        })),
    }
}

// Update webhook
pub async fn update_webhook(
    manager: web::Data<WebhookManager>,
    path: web::Path<Uuid>,
    body: web::Json<WebhookUpdate>,
) -> HttpResponse {
    let id = path.into_inner();

    match manager.update(&id, body.into_inner()).await {
        Some(webhook) => HttpResponse::Ok().json(webhook),
        None => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Webhook not found"
        })),
    }
}

// Delete webhook
pub async fn delete_webhook(
    manager: web::Data<WebhookManager>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();

    if manager.delete(&id).await {
        HttpResponse::NoContent().finish()
    } else {
        HttpResponse::NotFound().json(serde_json::json!({
            "error": "Webhook not found"
        }))
    }
}

// Get deliveries
pub async fn get_webhook_deliveries(
    manager: web::Data<WebhookManager>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();
    let deliveries = manager.get_deliveries(&id).await;
    HttpResponse::Ok().json(deliveries)
}

// Test webhook (trigger manually)
pub async fn test_webhook(
    manager: web::Data<WebhookManager>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let id = path.into_inner();

    match manager.get(&id).await {
        Some(webhook) => {
            let event = WebhookEvent::new(
                "webhook.test",
                serde_json::json!({
                    "message": "This is a test event",
                    "webhook_id": webhook.id
                }),
            );

            manager.trigger(event).await;

            HttpResponse::Ok().json(serde_json::json!({
                "message": "Test event triggered"
            }))
        }
        None => HttpResponse::NotFound().json(serde_json::json!({
            "error": "Webhook not found"
        })),
    }
}

// Main application
#[actix_web::main]
pub async fn main() -> std::io::Result<()> {
    use actix_web::{App, HttpServer};

    let manager = web::Data::new(WebhookManager::new());

    // Demo: สร้าง test webhook
    manager.register(Webhook::new(
        "Test Webhook",
        "https://webhook.site/test",
        vec!["order.*".to_string(), "user.*".to_string()],
    )).await;

    HttpServer::new(move || {
        App::new()
            .app_data(manager.clone())
            // Management API
            .route("/api/webhooks", web::post().to(create_webhook))
            .route("/api/webhooks", web::get().to(list_webhooks))
            .route("/api/webhooks/{id}", web::get().to(get_webhook))
            .route("/api/webhooks/{id}", web::patch().to(update_webhook))
            .route("/api/webhooks/{id}", web::delete().to(delete_webhook))
            .route("/api/webhooks/{id}/deliveries", web::get().to(get_webhook_deliveries))
            .route("/api/webhooks/{id}/test", web::post().to(test_webhook))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 8. ทดสอบ Webhook

```bash
# 1. สร้าง webhook
curl -X POST http://localhost:8080/api/webhooks \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Webhook",
    "url": "https://webhook.site/xxx",
    "events": ["order.*", "payment.completed"]
  }'

# 2. ดู webhooks ทั้งหมด
curl http://localhost:8080/api/webhooks

# 3. ทดสอบ webhook
curl -X POST http://localhost:8080/api/webhooks/{id}/test

# 4. ดู deliveries
curl http://localhost:8080/api/webhooks/{id}/deliveries

# 5. อัพเดต webhook
curl -X PATCH http://localhost:8080/api/webhooks/{id} \
  -H "Content-Type: application/json" \
  -d '{"active": false}'

# 6. ลบ webhook
curl -X DELETE http://localhost:8080/api/webhooks/{id}
```

---

## สรุป

✅ Webhook design patterns  
✅ HMAC-SHA256 signature generation และ verification  
✅ Retry ด้วย exponential backoff  
✅ Webhook delivery tracking  
✅ Webhook management API (CRUD)  
✅ Event filtering ด้วย patterns  
✅ Idempotency ด้วย event ID tracking  
✅ Incoming webhook receiver  

### Exercise

1. เพิ่ม webhook rotation (secret rotate API)
2. สร้าง webhook analytics dashboard
3. Implement circuit breaker - ปิด webhook ที่ fail บ่อยเกินไป
4. เพิ่ม event replay functionality

---

*[← Part 059: Message Queue with RabbitMQ](../part_059/README.md) | [Part 061: Caching with Redis →](../part_061/README.md)*
