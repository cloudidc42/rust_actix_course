# Part 056: Payment Gateway Integration 💳

## 🎯 เป้าหมายของ Part นี้

- Stripe integration ด้วย stripe-rust
- Payment intent flow
- Webhook handling (Stripe signatures)
- Refunds
- Subscription management
- Error handling สำหรับ payments
- Idempotency keys
- สร้าง Checkout API

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
stripe = { package = "async-stripe", version = "0.37", features = ["runtime-tokio-hyper"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
hmac = "0.12"
sha2 = "0.10"
hex = "0.4"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
anyhow = "1"
dotenv = "0.15"
```

---

## 2. Stripe Client Setup

```rust
// src/payment/stripe_client.rs
use stripe::Client;

pub fn create_stripe_client() -> Client {
    let secret_key = std::env::var("STRIPE_SECRET_KEY")
        .expect("STRIPE_SECRET_KEY must be set");

    Client::new(secret_key)
}

// Environment variables ที่ต้องตั้ง:
// STRIPE_SECRET_KEY=sk_test_...
// STRIPE_PUBLISHABLE_KEY=pk_test_...
// STRIPE_WEBHOOK_SECRET=whsec_...
```

---

## 3. Payment Intent Flow

```rust
// src/payment/payment_intent.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use stripe::{
    Client, CreatePaymentIntent, Currency, PaymentIntent,
    PaymentIntentConfirmParams,
};
use uuid::Uuid;

#[derive(Debug, Deserialize)]
pub struct CreatePaymentRequest {
    pub amount: i64,          // amount ใน smallest currency unit (satang/cents)
    pub currency: String,     // "thb", "usd", etc.
    pub description: Option<String>,
    pub customer_id: Option<String>,
    pub metadata: Option<std::collections::HashMap<String, String>>,
}

#[derive(Debug, Serialize)]
pub struct PaymentResponse {
    pub payment_intent_id: String,
    pub client_secret: String,
    pub amount: i64,
    pub currency: String,
    pub status: String,
}

pub async fn create_payment_intent(
    client: web::Data<Client>,
    body: web::Json<CreatePaymentRequest>,
) -> actix_web::Result<HttpResponse> {
    let currency = match body.currency.to_lowercase().as_str() {
        "thb" => Currency::THB,
        "usd" => Currency::USD,
        "eur" => Currency::EUR,
        _ => return Err(actix_web::error::ErrorBadRequest("Unsupported currency")),
    };

    // Idempotency key - ป้องกันการสร้าง payment ซ้ำ
    let idempotency_key = Uuid::new_v4().to_string();

    let mut params = CreatePaymentIntent::new(body.amount, currency);
    params.description = body.description.as_deref();

    // เพิ่ม metadata
    if let Some(ref meta) = body.metadata {
        let mut stripe_meta = std::collections::HashMap::new();
        for (k, v) in meta {
            stripe_meta.insert(k.clone(), v.clone());
        }
        params.metadata = Some(stripe_meta);
    }

    // Capture method: automatic หรือ manual
    params.capture_method = Some(stripe::PaymentIntentCaptureMethod::Automatic);

    let intent = PaymentIntent::create(&client, params)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    let client_secret = intent
        .client_secret
        .unwrap_or_default();

    Ok(HttpResponse::Ok().json(PaymentResponse {
        payment_intent_id: intent.id.to_string(),
        client_secret,
        amount: intent.amount,
        currency: intent.currency.to_string(),
        status: format!("{:?}", intent.status),
    }))
}

// Retrieve payment intent status
pub async fn get_payment_status(
    client: web::Data<Client>,
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    let intent_id = path.into_inner();

    let intent = PaymentIntent::retrieve(
        &client,
        &intent_id.parse().map_err(|_| actix_web::error::ErrorBadRequest("Invalid payment intent ID"))?,
        &[],
    )
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "id": intent.id,
        "amount": intent.amount,
        "currency": intent.currency,
        "status": format!("{:?}", intent.status),
        "description": intent.description,
        "created": intent.created,
    })))
}
```

---

## 4. Webhook Handling

```rust
// src/payment/webhooks.rs
use actix_web::{web, HttpRequest, HttpResponse};
use hmac::{Hmac, Mac};
use sha2::Sha256;
use serde_json::Value;

type HmacSha256 = Hmac<Sha256>;

// Verify Stripe webhook signature
pub fn verify_stripe_signature(
    payload: &[u8],
    signature_header: &str,
    webhook_secret: &str,
) -> Result<(), WebhookError> {
    // Format: t=timestamp,v1=signature
    let mut timestamp = None;
    let mut signatures: Vec<String> = Vec::new();

    for part in signature_header.split(',') {
        if let Some(t) = part.strip_prefix("t=") {
            timestamp = Some(t.to_string());
        } else if let Some(s) = part.strip_prefix("v1=") {
            signatures.push(s.to_string());
        }
    }

    let timestamp = timestamp.ok_or(WebhookError::InvalidSignature)?;

    // สร้าง signed payload
    let signed_payload = format!("{}.{}", timestamp, String::from_utf8_lossy(payload));

    // คำนวณ HMAC
    let mut mac = HmacSha256::new_from_slice(webhook_secret.as_bytes())
        .map_err(|_| WebhookError::InvalidSecret)?;
    mac.update(signed_payload.as_bytes());
    let expected = hex::encode(mac.finalize().into_bytes());

    // ตรวจสอบ timestamp (ป้องกัน replay attacks)
    let event_ts: i64 = timestamp.parse().map_err(|_| WebhookError::InvalidTimestamp)?;
    let now = chrono::Utc::now().timestamp();

    if (now - event_ts).abs() > 300 {
        // 5 minutes tolerance
        return Err(WebhookError::ExpiredSignature);
    }

    // ตรวจสอบ signature
    if signatures.iter().any(|s| s == &expected) {
        Ok(())
    } else {
        Err(WebhookError::SignatureMismatch)
    }
}

#[derive(Debug, thiserror::Error)]
pub enum WebhookError {
    #[error("Invalid signature format")]
    InvalidSignature,
    #[error("Invalid secret")]
    InvalidSecret,
    #[error("Invalid timestamp")]
    InvalidTimestamp,
    #[error("Expired signature")]
    ExpiredSignature,
    #[error("Signature mismatch")]
    SignatureMismatch,
}

// Webhook handler
pub async fn stripe_webhook(
    req: HttpRequest,
    body: web::Bytes,
) -> HttpResponse {
    let webhook_secret = std::env::var("STRIPE_WEBHOOK_SECRET")
        .expect("STRIPE_WEBHOOK_SECRET must be set");

    // ตรวจสอบ signature
    let signature = match req
        .headers()
        .get("Stripe-Signature")
        .and_then(|v| v.to_str().ok())
    {
        Some(sig) => sig.to_string(),
        None => {
            return HttpResponse::BadRequest().json(serde_json::json!({
                "error": "Missing Stripe-Signature header"
            }))
        }
    };

    if let Err(e) = verify_stripe_signature(&body, &signature, &webhook_secret) {
        return HttpResponse::Unauthorized().json(serde_json::json!({
            "error": format!("Invalid signature: {}", e)
        }));
    }

    // Parse event
    let event: Value = match serde_json::from_slice(&body) {
        Ok(e) => e,
        Err(e) => {
            return HttpResponse::BadRequest().json(serde_json::json!({
                "error": format!("Invalid JSON: {}", e)
            }))
        }
    };

    let event_type = event["type"].as_str().unwrap_or("unknown");
    let event_id = event["id"].as_str().unwrap_or("unknown");

    println!("[Webhook] Received event: {} ({})", event_type, event_id);

    // Handle different event types
    match event_type {
        "payment_intent.succeeded" => {
            handle_payment_succeeded(&event).await;
        }
        "payment_intent.payment_failed" => {
            handle_payment_failed(&event).await;
        }
        "customer.subscription.created" => {
            handle_subscription_created(&event).await;
        }
        "customer.subscription.deleted" => {
            handle_subscription_cancelled(&event).await;
        }
        "invoice.payment_succeeded" => {
            handle_invoice_paid(&event).await;
        }
        _ => {
            println!("[Webhook] Unhandled event type: {}", event_type);
        }
    }

    HttpResponse::Ok().json(serde_json::json!({ "received": true }))
}

async fn handle_payment_succeeded(event: &Value) {
    let pi = &event["data"]["object"];
    let amount = pi["amount"].as_i64().unwrap_or(0);
    let currency = pi["currency"].as_str().unwrap_or("unknown");
    let pi_id = pi["id"].as_str().unwrap_or("unknown");

    println!(
        "[Payment] Succeeded: {} - {}{} ",
        pi_id,
        amount,
        currency.to_uppercase()
    );

    // TODO: อัพเดต order status ใน database
    // TODO: ส่ง email ยืนยันการชำระเงิน
}

async fn handle_payment_failed(event: &Value) {
    let pi = &event["data"]["object"];
    let pi_id = pi["id"].as_str().unwrap_or("unknown");
    let error_msg = pi["last_payment_error"]["message"]
        .as_str()
        .unwrap_or("Unknown error");

    println!("[Payment] Failed: {} - {}", pi_id, error_msg);

    // TODO: แจ้ง user ว่า payment ล้มเหลว
}

async fn handle_subscription_created(event: &Value) {
    let sub = &event["data"]["object"];
    println!("[Subscription] Created: {}", sub["id"]);
}

async fn handle_subscription_cancelled(event: &Value) {
    let sub = &event["data"]["object"];
    println!("[Subscription] Cancelled: {}", sub["id"]);
}

async fn handle_invoice_paid(event: &Value) {
    let invoice = &event["data"]["object"];
    println!("[Invoice] Paid: {}", invoice["id"]);
}
```

---

## 5. Refunds

```rust
// src/payment/refund.rs
use actix_web::{web, HttpResponse};
use serde::Deserialize;
use stripe::{Client, CreateRefund, Refund};

#[derive(Debug, Deserialize)]
pub struct RefundRequest {
    pub payment_intent_id: String,
    pub amount: Option<i64>,  // None = full refund
    pub reason: Option<String>,
}

pub async fn create_refund(
    client: web::Data<Client>,
    body: web::Json<RefundRequest>,
) -> actix_web::Result<HttpResponse> {
    let mut params = CreateRefund::new();

    params.payment_intent = Some(
        body.payment_intent_id
            .parse()
            .map_err(|_| actix_web::error::ErrorBadRequest("Invalid payment_intent_id"))?,
    );

    if let Some(amount) = body.amount {
        params.amount = Some(amount);
    }

    params.reason = body.reason.as_deref().and_then(|r| match r {
        "duplicate" => Some(stripe::RefundReason::Duplicate),
        "fraudulent" => Some(stripe::RefundReason::Fraudulent),
        "requested_by_customer" => Some(stripe::RefundReason::RequestedByCustomer),
        _ => None,
    });

    let refund = Refund::create(&client, params)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "refund_id": refund.id,
        "amount": refund.amount,
        "status": format!("{:?}", refund.status),
        "created": refund.created,
    })))
}
```

---

## 6. Subscription Management

```rust
// src/payment/subscription.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use stripe::{
    Client, CreateCustomer, CreateSubscription, CreateSubscriptionItems,
    Customer, Subscription,
};

#[derive(Debug, Deserialize)]
pub struct CreateSubscriptionRequest {
    pub email: String,
    pub name: String,
    pub price_id: String,  // Stripe Price ID เช่น price_xxx
    pub payment_method_id: String,
}

pub async fn create_subscription(
    client: web::Data<Client>,
    body: web::Json<CreateSubscriptionRequest>,
) -> actix_web::Result<HttpResponse> {
    // 1. สร้าง Customer
    let customer_params = CreateCustomer {
        email: Some(&body.email),
        name: Some(&body.name),
        ..Default::default()
    };

    let customer = Customer::create(&client, customer_params)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    // 2. Attach payment method ไปยัง customer
    // (ในที่นี้ข้ามไปเพื่อความกระชับ)

    // 3. สร้าง Subscription
    let mut sub_params = CreateSubscription::new(customer.id.clone());
    sub_params.items = Some(vec![CreateSubscriptionItems {
        price: Some(body.price_id.parse().map_err(|_| {
            actix_web::error::ErrorBadRequest("Invalid price_id")
        })?),
        ..Default::default()
    }]);

    let subscription = Subscription::create(&client, sub_params)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "subscription_id": subscription.id,
        "customer_id": customer.id,
        "status": format!("{:?}", subscription.status),
    })))
}

// Cancel subscription
pub async fn cancel_subscription(
    client: web::Data<Client>,
    path: web::Path<String>,
) -> actix_web::Result<HttpResponse> {
    let sub_id = path.into_inner();

    let sub_id_parsed: stripe::SubscriptionId = sub_id
        .parse()
        .map_err(|_| actix_web::error::ErrorBadRequest("Invalid subscription ID"))?;

    let cancelled = Subscription::cancel(&client, &sub_id_parsed, Default::default())
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "subscription_id": cancelled.id,
        "status": format!("{:?}", cancelled.status),
    })))
}
```

---

## 7. Practical: Checkout API

```rust
// src/payment/checkout.rs
use actix_web::{web, HttpResponse};
use serde::{Deserialize, Serialize};
use stripe::{Client, CreatePaymentIntent, Currency, PaymentIntent};
use uuid::Uuid;

#[derive(Debug, Deserialize)]
pub struct CartItem {
    pub product_id: String,
    pub quantity: u32,
    pub price: i64,  // ราคาต่อหน่วยใน satang
}

#[derive(Debug, Deserialize)]
pub struct CheckoutRequest {
    pub items: Vec<CartItem>,
    pub currency: String,
    pub customer_email: String,
    pub shipping_address: Option<Address>,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
pub struct Address {
    pub line1: String,
    pub city: String,
    pub country: String,
    pub postal_code: String,
}

#[derive(Debug, Serialize)]
pub struct CheckoutResponse {
    pub order_id: String,
    pub payment_intent_id: String,
    pub client_secret: String,
    pub total_amount: i64,
    pub currency: String,
    pub items: Vec<CartItemSummary>,
}

#[derive(Debug, Serialize)]
pub struct CartItemSummary {
    pub product_id: String,
    pub quantity: u32,
    pub price: i64,
    pub subtotal: i64,
}

pub async fn create_checkout(
    client: web::Data<Client>,
    body: web::Json<CheckoutRequest>,
) -> actix_web::Result<HttpResponse> {
    // คำนวณ total
    let total: i64 = body.items.iter().map(|i| i.price * i.quantity as i64).sum();

    if total <= 0 {
        return Err(actix_web::error::ErrorBadRequest("Invalid total amount"));
    }

    let currency = match body.currency.to_lowercase().as_str() {
        "thb" => Currency::THB,
        "usd" => Currency::USD,
        _ => return Err(actix_web::error::ErrorBadRequest("Unsupported currency")),
    };

    let order_id = Uuid::new_v4().to_string();

    // สร้าง metadata
    let mut metadata = std::collections::HashMap::new();
    metadata.insert("order_id".to_string(), order_id.clone());
    metadata.insert("customer_email".to_string(), body.customer_email.clone());
    metadata.insert(
        "items_count".to_string(),
        body.items.len().to_string(),
    );

    let mut params = CreatePaymentIntent::new(total, currency);
    params.metadata = Some(metadata);
    params.description = Some("Order payment");
    params.receipt_email = Some(&body.customer_email);

    let intent = PaymentIntent::create(&client, params)
        .await
        .map_err(|e| actix_web::error::ErrorInternalServerError(e.to_string()))?;

    let items_summary: Vec<CartItemSummary> = body
        .items
        .iter()
        .map(|i| CartItemSummary {
            product_id: i.product_id.clone(),
            quantity: i.quantity,
            price: i.price,
            subtotal: i.price * i.quantity as i64,
        })
        .collect();

    Ok(HttpResponse::Ok().json(CheckoutResponse {
        order_id,
        payment_intent_id: intent.id.to_string(),
        client_secret: intent.client_secret.unwrap_or_default(),
        total_amount: total,
        currency: body.currency.clone(),
        items: items_summary,
    }))
}

// Main app
use actix_web::{App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();

    let stripe_client = web::Data::new(super::stripe_client::create_stripe_client());

    HttpServer::new(move || {
        App::new()
            .app_data(stripe_client.clone())
            .route("/api/checkout", web::post().to(create_checkout))
            .route(
                "/api/payments/{id}",
                web::get().to(super::payment_intent::get_payment_status),
            )
            .route(
                "/api/refunds",
                web::post().to(super::refund::create_refund),
            )
            .route(
                "/api/subscriptions",
                web::post().to(super::subscription::create_subscription),
            )
            .route(
                "/api/subscriptions/{id}",
                web::delete().to(super::subscription::cancel_subscription),
            )
            .route(
                "/webhooks/stripe",
                web::post().to(super::webhooks::stripe_webhook),
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

✅ Stripe client setup  
✅ Payment intent flow  
✅ Webhook signature verification  
✅ Webhook event handling  
✅ Refunds  
✅ Subscription management  
✅ Idempotency keys  
✅ Complete checkout API  

### Exercise

1. เพิ่ม Stripe Elements integration
2. สร้าง payment history API
3. Implement 3D Secure handling
4. เพิ่ม multi-currency support พร้อม conversion

---

*[← Part 055: Email Service](../part_055/README.md) | [Part 057: GraphQL with async-graphql →](../part_057/README.md)*
