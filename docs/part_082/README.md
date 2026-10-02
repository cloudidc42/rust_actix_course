# Part 082: Project: E-commerce API 🛒

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้าง E-commerce API ที่สมบูรณ์ด้วย Rust และ Actix-web
- จัดการ Products, Categories, Orders, Cart
- ทำ Product Inventory Management
- สร้าง Order Processing Workflow
- ทำ Payment Integration แบบ Mock Stripe
- จัดการ Discount Codes
- คำนวณ Shipping
- สร้าง Admin Dashboard Endpoints

---

## 1. โครงสร้างโปรเจกต์

```
ecommerce_api/
├── Cargo.toml
├── .env
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_categories.sql
│   ├── 003_create_products.sql
│   ├── 004_create_cart.sql
│   ├── 005_create_orders.sql
│   └── 006_create_discounts.sql
└── src/
    ├── main.rs
    ├── errors.rs
    ├── models/
    │   ├── mod.rs
    │   ├── product.rs
    │   ├── category.rs
    │   ├── cart.rs
    │   ├── order.rs
    │   └── discount.rs
    ├── handlers/
    │   ├── mod.rs
    │   ├── products.rs
    │   ├── categories.rs
    │   ├── cart.rs
    │   ├── orders.rs
    │   ├── payments.rs
    │   └── admin.rs
    └── services/
        ├── mod.rs
        ├── inventory.rs
        ├── payment.rs
        ├── shipping.rs
        └── notification.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "ecommerce_api"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-cors = "0.7"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono", "rust_decimal"] }
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
rust_decimal = { version = "1", features = ["serde-with-str"] }
rust_decimal_macros = "1"
jsonwebtoken = "9"
bcrypt = "0.15"
dotenv = "0.15"
env_logger = "0.11"
log = "0.4"
validator = { version = "0.18", features = ["derive"] }
thiserror = "1"
reqwest = { version = "0.12", features = ["json"] }
rand = "0.8"
```

---

## 3. Models

### 3.1 Product Model (`src/models/product.rs`)

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "product_status", rename_all = "lowercase")]
pub enum ProductStatus {
    Active,
    Inactive,
    OutOfStock,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Product {
    pub id: Uuid,
    pub name: String,
    pub slug: String,
    pub description: Option<String>,
    pub price: Decimal,
    pub compare_price: Option<Decimal>,
    pub cost_price: Option<Decimal>,
    pub sku: String,
    pub barcode: Option<String>,
    pub category_id: Option<Uuid>,
    pub stock_quantity: i32,
    pub low_stock_threshold: i32,
    pub weight: Option<Decimal>,
    pub images: Vec<String>,
    pub tags: Vec<String>,
    pub status: ProductStatus,
    pub is_featured: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateProductRequest {
    #[validate(length(min = 2, max = 200))]
    pub name: String,
    pub description: Option<String>,
    #[validate(range(min = 0.01))]
    pub price: Decimal,
    pub compare_price: Option<Decimal>,
    pub cost_price: Option<Decimal>,
    #[validate(length(min = 1, max = 50))]
    pub sku: String,
    pub category_id: Option<Uuid>,
    pub stock_quantity: i32,
    pub low_stock_threshold: Option<i32>,
    pub weight: Option<Decimal>,
    pub images: Option<Vec<String>>,
    pub tags: Option<Vec<String>>,
}

#[derive(Debug, Deserialize)]
pub struct ProductQuery {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
    pub search: Option<String>,
    pub category_id: Option<Uuid>,
    pub min_price: Option<Decimal>,
    pub max_price: Option<Decimal>,
    pub in_stock: Option<bool>,
    pub is_featured: Option<bool>,
    pub sort: Option<String>,
}

#[derive(Debug, Deserialize)]
pub struct UpdateStockRequest {
    pub quantity_change: i32,
    pub reason: Option<String>,
}
```

### 3.2 Cart Model (`src/models/cart.rs`)

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct CartItem {
    pub id: Uuid,
    pub user_id: Uuid,
    pub product_id: Uuid,
    pub quantity: i32,
    pub price_at_time: Decimal,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize)]
pub struct CartSummary {
    pub items: Vec<CartItemDetail>,
    pub subtotal: Decimal,
    pub discount_amount: Decimal,
    pub shipping_amount: Decimal,
    pub tax_amount: Decimal,
    pub total: Decimal,
    pub discount_code: Option<String>,
    pub item_count: i32,
}

#[derive(Debug, Serialize)]
pub struct CartItemDetail {
    pub id: Uuid,
    pub product_id: Uuid,
    pub product_name: String,
    pub product_image: Option<String>,
    pub sku: String,
    pub quantity: i32,
    pub unit_price: Decimal,
    pub total_price: Decimal,
    pub stock_available: i32,
}

#[derive(Debug, Deserialize)]
pub struct AddToCartRequest {
    pub product_id: Uuid,
    pub quantity: i32,
}

#[derive(Debug, Deserialize)]
pub struct UpdateCartItemRequest {
    pub quantity: i32,
}
```

### 3.3 Order Model (`src/models/order.rs`)

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use validator::Validate;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "order_status", rename_all = "lowercase")]
pub enum OrderStatus {
    Pending,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type, PartialEq)]
#[sqlx(type_name = "payment_status", rename_all = "lowercase")]
pub enum PaymentStatus {
    Pending,
    Paid,
    Failed,
    Refunded,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct Order {
    pub id: Uuid,
    pub order_number: String,
    pub user_id: Uuid,
    pub status: OrderStatus,
    pub payment_status: PaymentStatus,
    pub payment_method: Option<String>,
    pub payment_intent_id: Option<String>,
    pub subtotal: Decimal,
    pub discount_amount: Decimal,
    pub shipping_amount: Decimal,
    pub tax_amount: Decimal,
    pub total: Decimal,
    pub discount_code: Option<String>,
    pub shipping_address: serde_json::Value,
    pub billing_address: Option<serde_json::Value>,
    pub notes: Option<String>,
    pub shipped_at: Option<DateTime<Utc>>,
    pub delivered_at: Option<DateTime<Utc>>,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct OrderItem {
    pub id: Uuid,
    pub order_id: Uuid,
    pub product_id: Uuid,
    pub product_name: String,
    pub product_sku: String,
    pub quantity: i32,
    pub unit_price: Decimal,
    pub total_price: Decimal,
}

#[derive(Debug, Deserialize, Validate)]
pub struct CreateOrderRequest {
    pub shipping_address: ShippingAddress,
    pub billing_address: Option<ShippingAddress>,
    pub discount_code: Option<String>,
    pub notes: Option<String>,
}

#[derive(Debug, Serialize, Deserialize, Validate)]
pub struct ShippingAddress {
    #[validate(length(min = 1))]
    pub full_name: String,
    pub phone: String,
    pub address_line1: String,
    pub address_line2: Option<String>,
    pub city: String,
    pub state: String,
    pub postal_code: String,
    pub country: String,
}
```

### 3.4 Discount Model (`src/models/discount.rs`)

```rust
use chrono::{DateTime, Utc};
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::Type)]
#[sqlx(type_name = "discount_type", rename_all = "lowercase")]
pub enum DiscountType {
    Percentage,
    FixedAmount,
}

#[derive(Debug, Clone, Serialize, Deserialize, FromRow)]
pub struct DiscountCode {
    pub id: Uuid,
    pub code: String,
    pub discount_type: DiscountType,
    pub discount_value: Decimal,
    pub minimum_order_amount: Option<Decimal>,
    pub maximum_uses: Option<i32>,
    pub current_uses: i32,
    pub expires_at: Option<DateTime<Utc>>,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct ApplyDiscountRequest {
    pub code: String,
}
```

---

## 4. Services

### 4.1 Inventory Service (`src/services/inventory.rs`)

```rust
use rust_decimal::Decimal;
use sqlx::PgPool;
use uuid::Uuid;

use crate::errors::AppError;

pub struct InventoryService;

impl InventoryService {
    pub async fn check_availability(
        pool: &PgPool,
        product_id: Uuid,
        quantity: i32,
    ) -> Result<bool, AppError> {
        let stock: i32 = sqlx::query_scalar!(
            "SELECT stock_quantity FROM products WHERE id = $1 AND status = 'active'",
            product_id
        )
        .fetch_optional(pool)
        .await?
        .ok_or_else(|| AppError::NotFound("Product not found".to_string()))?;

        Ok(stock >= quantity)
    }

    pub async fn reserve_stock(
        pool: &PgPool,
        product_id: Uuid,
        quantity: i32,
    ) -> Result<(), AppError> {
        let result = sqlx::query!(
            r#"
            UPDATE products
            SET stock_quantity = stock_quantity - $1,
                updated_at = NOW()
            WHERE id = $2 AND stock_quantity >= $1
            "#,
            quantity,
            product_id
        )
        .execute(pool)
        .await?;

        if result.rows_affected() == 0 {
            return Err(AppError::BadRequest("Insufficient stock".to_string()));
        }

        // Update status if out of stock
        sqlx::query!(
            r#"
            UPDATE products
            SET status = 'out_of_stock'
            WHERE id = $1 AND stock_quantity = 0
            "#,
            product_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    pub async fn release_stock(
        pool: &PgPool,
        product_id: Uuid,
        quantity: i32,
    ) -> Result<(), AppError> {
        sqlx::query!(
            r#"
            UPDATE products
            SET stock_quantity = stock_quantity + $1,
                status = CASE WHEN stock_quantity + $1 > 0 THEN 'active' ELSE status END,
                updated_at = NOW()
            WHERE id = $2
            "#,
            quantity,
            product_id
        )
        .execute(pool)
        .await?;

        Ok(())
    }

    pub async fn get_low_stock_products(
        pool: &PgPool,
    ) -> Result<Vec<LowStockProduct>, AppError> {
        let products = sqlx::query_as!(
            LowStockProduct,
            r#"
            SELECT id, name, sku, stock_quantity, low_stock_threshold
            FROM products
            WHERE stock_quantity <= low_stock_threshold AND status = 'active'
            ORDER BY stock_quantity ASC
            "#
        )
        .fetch_all(pool)
        .await?;

        Ok(products)
    }
}

#[derive(Debug, serde::Serialize)]
pub struct LowStockProduct {
    pub id: Uuid,
    pub name: String,
    pub sku: String,
    pub stock_quantity: i32,
    pub low_stock_threshold: i32,
}
```

### 4.2 Payment Service - Mock Stripe (`src/services/payment.rs`)

```rust
use rust_decimal::Decimal;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

use crate::errors::AppError;

#[derive(Debug, Serialize, Deserialize)]
pub struct PaymentIntent {
    pub id: String,
    pub amount: i64,
    pub currency: String,
    pub status: PaymentIntentStatus,
    pub client_secret: String,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum PaymentIntentStatus {
    RequiresPaymentMethod,
    RequiresConfirmation,
    Processing,
    Succeeded,
    Canceled,
}

#[derive(Debug, Deserialize)]
pub struct ConfirmPaymentRequest {
    pub payment_intent_id: String,
    pub payment_method: PaymentMethod,
}

#[derive(Debug, Deserialize)]
pub struct PaymentMethod {
    pub card_number: String,
    pub exp_month: u32,
    pub exp_year: u32,
    pub cvc: String,
}

pub struct MockStripeService {
    api_key: String,
}

impl MockStripeService {
    pub fn new(api_key: String) -> Self {
        MockStripeService { api_key }
    }

    pub async fn create_payment_intent(
        &self,
        amount: Decimal,
        currency: &str,
        order_id: Uuid,
    ) -> Result<PaymentIntent, AppError> {
        // Mock Stripe payment intent creation
        let amount_cents = (amount * Decimal::from(100))
            .to_string()
            .parse::<i64>()
            .unwrap_or(0);

        let intent_id = format!("pi_{}", uuid::Uuid::new_v4().to_string().replace("-", ""));
        let client_secret = format!("{}_secret_{}", intent_id, uuid::Uuid::new_v4().to_string().replace("-", ""));

        log::info!(
            "Created mock payment intent {} for order {} amount {}{}",
            intent_id, order_id, amount, currency
        );

        Ok(PaymentIntent {
            id: intent_id,
            amount: amount_cents,
            currency: currency.to_string(),
            status: PaymentIntentStatus::RequiresPaymentMethod,
            client_secret,
        })
    }

    pub async fn confirm_payment(
        &self,
        payment_intent_id: &str,
        payment_method: &PaymentMethod,
    ) -> Result<PaymentIntent, AppError> {
        // Mock payment confirmation
        // In real implementation, this would call Stripe API
        let is_valid_card = payment_method.card_number.len() == 16 
            && payment_method.card_number.chars().all(|c| c.is_ascii_digit());

        // Simulate declined card for test number
        if payment_method.card_number == "4000000000000002" {
            return Err(AppError::BadRequest("Card declined".to_string()));
        }

        if !is_valid_card {
            return Err(AppError::BadRequest("Invalid card number".to_string()));
        }

        let client_secret = format!("{}_secret_confirmed", payment_intent_id);

        Ok(PaymentIntent {
            id: payment_intent_id.to_string(),
            amount: 0,
            currency: "usd".to_string(),
            status: PaymentIntentStatus::Succeeded,
            client_secret,
        })
    }

    pub async fn refund_payment(&self, payment_intent_id: &str) -> Result<(), AppError> {
        // Mock refund processing
        log::info!("Processing mock refund for payment intent {}", payment_intent_id);
        Ok(())
    }
}
```

### 4.3 Shipping Service (`src/services/shipping.rs`)

```rust
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct ShippingRate {
    pub method: ShippingMethod,
    pub name: String,
    pub description: String,
    pub price: Decimal,
    pub estimated_days: u8,
}

#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum ShippingMethod {
    Standard,
    Express,
    Overnight,
    Free,
}

pub struct ShippingService;

impl ShippingService {
    pub fn calculate_rates(
        subtotal: Decimal,
        weight_kg: Option<Decimal>,
        destination_country: &str,
    ) -> Vec<ShippingRate> {
        let mut rates = Vec::new();
        let is_domestic = destination_country == "TH";

        // Free shipping for orders over 1000 THB (or $30 for international)
        let free_threshold = if is_domestic { dec!(1000) } else { dec!(30) };

        if subtotal >= free_threshold {
            rates.push(ShippingRate {
                method: ShippingMethod::Free,
                name: "Free Shipping".to_string(),
                description: "Free standard shipping".to_string(),
                price: dec!(0),
                estimated_days: if is_domestic { 5 } else { 14 },
            });
        }

        if is_domestic {
            rates.push(ShippingRate {
                method: ShippingMethod::Standard,
                name: "Standard Delivery".to_string(),
                description: "Delivered in 3-5 business days".to_string(),
                price: dec!(50),
                estimated_days: 5,
            });
            rates.push(ShippingRate {
                method: ShippingMethod::Express,
                name: "Express Delivery".to_string(),
                description: "Delivered in 1-2 business days".to_string(),
                price: dec!(120),
                estimated_days: 2,
            });
            rates.push(ShippingRate {
                method: ShippingMethod::Overnight,
                name: "Overnight Delivery".to_string(),
                description: "Next business day delivery".to_string(),
                price: dec!(250),
                estimated_days: 1,
            });
        } else {
            // International shipping - weight based
            let base_weight = weight_kg.unwrap_or(dec!(0.5));
            let international_rate = dec!(15) + (base_weight * dec!(10));

            rates.push(ShippingRate {
                method: ShippingMethod::Standard,
                name: "International Standard".to_string(),
                description: "Delivered in 10-14 business days".to_string(),
                price: international_rate,
                estimated_days: 14,
            });
            rates.push(ShippingRate {
                method: ShippingMethod::Express,
                name: "International Express".to_string(),
                description: "Delivered in 3-5 business days".to_string(),
                price: international_rate * dec!(2.5),
                estimated_days: 5,
            });
        }

        rates
    }

    pub fn calculate_tax(subtotal: Decimal, country: &str) -> Decimal {
        let tax_rate = match country {
            "TH" => dec!(0.07),   // 7% VAT
            "US" => dec!(0.08),   // ~8% average
            "GB" => dec!(0.20),   // 20% VAT
            "DE" | "FR" => dec!(0.19), // 19% VAT
            _ => dec!(0),
        };
        subtotal * tax_rate
    }
}
```

---

## 5. Handlers

### 5.1 Products Handler (`src/handlers/products.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use rust_decimal::Decimal;
use slug::slugify;
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::models::product::{CreateProductRequest, ProductQuery, UpdateStockRequest};

pub async fn list_products(
    pool: web::Data<PgPool>,
    query: web::Query<ProductQuery>,
) -> Result<HttpResponse, AppError> {
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(20).min(100);
    let offset = ((page - 1) * per_page) as i64;

    let products = sqlx::query!(
        r#"
        SELECT id, name, slug, description, price, compare_price,
               sku, category_id, stock_quantity, images, status::text,
               is_featured, created_at
        FROM products
        WHERE ($1::text IS NULL OR name ILIKE '%' || $1 || '%' OR description ILIKE '%' || $1 || '%')
          AND ($2::uuid IS NULL OR category_id = $2)
          AND ($3::numeric IS NULL OR price >= $3)
          AND ($4::numeric IS NULL OR price <= $4)
          AND ($5::bool IS NULL OR ($5 = true AND stock_quantity > 0))
          AND ($6::bool IS NULL OR is_featured = $6)
          AND status = 'active'
        ORDER BY
            CASE WHEN $7 = 'price_asc' THEN price END ASC,
            CASE WHEN $7 = 'price_desc' THEN price END DESC,
            CASE WHEN $7 = 'newest' THEN created_at END DESC,
            is_featured DESC, created_at DESC
        LIMIT $8 OFFSET $9
        "#,
        query.search,
        query.category_id,
        query.min_price as Option<Decimal>,
        query.max_price as Option<Decimal>,
        query.in_stock,
        query.is_featured,
        query.sort,
        per_page as i64,
        offset
    )
    .fetch_all(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "products": products,
        "page": page,
        "per_page": per_page,
    })))
}

pub async fn create_product(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    body: web::Json<CreateProductRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;

    let product_id = Uuid::new_v4();
    let slug = slugify(&body.name);
    let images = body.images.clone().unwrap_or_default();
    let tags = body.tags.clone().unwrap_or_default();

    sqlx::query!(
        r#"
        INSERT INTO products (id, name, slug, description, price, compare_price,
                             cost_price, sku, barcode, category_id, stock_quantity,
                             low_stock_threshold, weight, images, tags)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15)
        "#,
        product_id,
        body.name,
        slug,
        body.description,
        body.price,
        body.compare_price,
        body.cost_price,
        body.sku,
        None::<String>,
        body.category_id,
        body.stock_quantity,
        body.low_stock_threshold.unwrap_or(5),
        body.weight,
        &images as &[String],
        &tags as &[String]
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Created().json(serde_json::json!({
        "id": product_id,
        "message": "Product created successfully"
    })))
}

pub async fn update_stock(
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
    body: web::Json<UpdateStockRequest>,
) -> Result<HttpResponse, AppError> {
    let product_id = path.into_inner();

    sqlx::query!(
        r#"
        UPDATE products
        SET stock_quantity = stock_quantity + $1,
            updated_at = NOW()
        WHERE id = $2
        "#,
        body.quantity_change,
        product_id
    )
    .execute(pool.get_ref())
    .await?;

    let new_stock: i32 = sqlx::query_scalar!(
        "SELECT stock_quantity FROM products WHERE id = $1",
        product_id
    )
    .fetch_one(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "product_id": product_id,
        "new_stock_quantity": new_stock,
        "reason": body.reason,
    })))
}
```

### 5.2 Orders Handler (`src/handlers/orders.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use rust_decimal::Decimal;
use rust_decimal_macros::dec;
use sqlx::PgPool;
use uuid::Uuid;
use validator::Validate;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::order::{CreateOrderRequest, OrderStatus};
use crate::services::inventory::InventoryService;
use crate::services::payment::MockStripeService;
use crate::services::shipping::ShippingService;

pub async fn create_order(
    pool: web::Data<PgPool>,
    stripe: web::Data<MockStripeService>,
    req: HttpRequest,
    body: web::Json<CreateOrderRequest>,
) -> Result<HttpResponse, AppError> {
    body.validate()?;
    let claims = require_auth(&req)?;
    let user_id = claims.sub;

    // Get cart items
    let cart_items = sqlx::query!(
        r#"
        SELECT ci.id, ci.product_id, ci.quantity, ci.price_at_time,
               p.name, p.sku, p.stock_quantity, p.weight
        FROM cart_items ci
        JOIN products p ON ci.product_id = p.id
        WHERE ci.user_id = $1
        "#,
        user_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    if cart_items.is_empty() {
        return Err(AppError::BadRequest("Cart is empty".to_string()));
    }

    // Verify stock availability
    for item in &cart_items {
        InventoryService::check_availability(pool.get_ref(), item.product_id, item.quantity).await?;
    }

    // Calculate totals
    let subtotal: Decimal = cart_items
        .iter()
        .map(|i| i.price_at_time * Decimal::from(i.quantity))
        .sum();

    // Apply discount code
    let discount_amount = if let Some(code) = &body.discount_code {
        apply_discount_code(pool.get_ref(), code, subtotal).await?
    } else {
        dec!(0)
    };

    // Calculate shipping
    let total_weight: Decimal = cart_items
        .iter()
        .filter_map(|i| i.weight)
        .sum();

    let shipping_rates = ShippingService::calculate_rates(
        subtotal,
        Some(total_weight),
        &body.shipping_address.country,
    );
    let shipping_amount = shipping_rates.first()
        .map(|r| r.price)
        .unwrap_or(dec!(0));

    let tax_amount = ShippingService::calculate_tax(subtotal, &body.shipping_address.country);
    let total = subtotal - discount_amount + shipping_amount + tax_amount;

    // Create order
    let order_id = Uuid::new_v4();
    let order_number = format!("ORD-{}", &order_id.to_string()[..8].to_uppercase());

    let shipping_json = serde_json::to_value(&body.shipping_address)
        .map_err(|e| AppError::InternalError(e.to_string()))?;

    let mut tx = pool.begin().await?;

    sqlx::query!(
        r#"
        INSERT INTO orders (id, order_number, user_id, subtotal, discount_amount,
                           shipping_amount, tax_amount, total, discount_code, shipping_address)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        "#,
        order_id,
        order_number,
        user_id,
        subtotal,
        discount_amount,
        shipping_amount,
        tax_amount,
        total,
        body.discount_code,
        shipping_json
    )
    .execute(&mut *tx)
    .await?;

    // Create order items and reserve stock
    for item in &cart_items {
        sqlx::query!(
            r#"
            INSERT INTO order_items (id, order_id, product_id, product_name, product_sku, quantity, unit_price, total_price)
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            "#,
            Uuid::new_v4(),
            order_id,
            item.product_id,
            item.name,
            item.sku,
            item.quantity,
            item.price_at_time,
            item.price_at_time * Decimal::from(item.quantity)
        )
        .execute(&mut *tx)
        .await?;
    }

    // Clear cart
    sqlx::query!("DELETE FROM cart_items WHERE user_id = $1", user_id)
        .execute(&mut *tx)
        .await?;

    tx.commit().await?;

    // Reserve stock (after commit)
    for item in &cart_items {
        InventoryService::reserve_stock(pool.get_ref(), item.product_id, item.quantity).await?;
    }

    // Create payment intent
    let payment_intent = stripe.create_payment_intent(total, "thb", order_id).await?;

    // Update order with payment intent
    sqlx::query!(
        "UPDATE orders SET payment_intent_id = $1 WHERE id = $2",
        payment_intent.id,
        order_id
    )
    .execute(pool.get_ref())
    .await?;

    Ok(HttpResponse::Created().json(serde_json::json!({
        "order_id": order_id,
        "order_number": order_number,
        "total": total,
        "payment_intent": {
            "id": payment_intent.id,
            "client_secret": payment_intent.client_secret,
        }
    })))
}

async fn apply_discount_code(
    pool: &PgPool,
    code: &str,
    subtotal: Decimal,
) -> Result<Decimal, AppError> {
    let discount = sqlx::query!(
        r#"
        SELECT discount_type::text, discount_value, minimum_order_amount, maximum_uses, current_uses
        FROM discount_codes
        WHERE code = $1 AND is_active = true
          AND (expires_at IS NULL OR expires_at > NOW())
        "#,
        code.to_uppercase()
    )
    .fetch_optional(pool)
    .await?
    .ok_or_else(|| AppError::BadRequest("Invalid or expired discount code".to_string()))?;

    if let Some(min_amount) = discount.minimum_order_amount {
        if subtotal < min_amount {
            return Err(AppError::BadRequest(
                format!("Minimum order amount for this code is {}", min_amount)
            ));
        }
    }

    if let Some(max_uses) = discount.maximum_uses {
        if discount.current_uses >= max_uses {
            return Err(AppError::BadRequest("Discount code has reached maximum uses".to_string()));
        }
    }

    let discount_amount = match discount.discount_type.as_deref() {
        Some("percentage") => subtotal * (discount.discount_value / Decimal::from(100)),
        Some("fixed_amount") => discount.discount_value.min(subtotal),
        _ => dec!(0),
    };

    // Increment usage
    sqlx::query!(
        "UPDATE discount_codes SET current_uses = current_uses + 1 WHERE code = $1",
        code.to_uppercase()
    )
    .execute(pool)
    .await?;

    Ok(discount_amount)
}

pub async fn get_order(
    pool: web::Data<PgPool>,
    req: HttpRequest,
    path: web::Path<Uuid>,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    let order_id = path.into_inner();

    let order = sqlx::query_as!(
        crate::models::order::Order,
        r#"
        SELECT id, order_number, user_id, status as "status: OrderStatus",
               payment_status as "payment_status: crate::models::order::PaymentStatus",
               payment_method, payment_intent_id, subtotal, discount_amount,
               shipping_amount, tax_amount, total, discount_code,
               shipping_address, billing_address, notes,
               shipped_at, delivered_at, created_at, updated_at
        FROM orders WHERE id = $1 AND user_id = $2
        "#,
        order_id,
        claims.sub
    )
    .fetch_optional(pool.get_ref())
    .await?
    .ok_or_else(|| AppError::NotFound("Order not found".to_string()))?;

    let items = sqlx::query_as!(
        crate::models::order::OrderItem,
        "SELECT * FROM order_items WHERE order_id = $1",
        order_id
    )
    .fetch_all(pool.get_ref())
    .await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "order": order,
        "items": items,
    })))
}

use rust_decimal_macros::dec;
```

### 5.3 Admin Dashboard (`src/handlers/admin.rs`)

```rust
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;

use crate::errors::AppError;
use crate::middleware::auth::require_auth;
use crate::models::user::UserRole;
use crate::services::inventory::InventoryService;

pub async fn dashboard_stats(
    pool: web::Data<PgPool>,
    req: HttpRequest,
) -> Result<HttpResponse, AppError> {
    let claims = require_auth(&req)?;
    if claims.role != UserRole::Admin {
        return Err(AppError::Forbidden("Admin access required".to_string()));
    }

    // Revenue stats
    let revenue = sqlx::query!(
        r#"
        SELECT
            SUM(total) FILTER (WHERE created_at >= date_trunc('day', NOW())) as today_revenue,
            SUM(total) FILTER (WHERE created_at >= date_trunc('month', NOW())) as month_revenue,
            SUM(total) as total_revenue,
            COUNT(*) FILTER (WHERE created_at >= date_trunc('day', NOW())) as today_orders,
            COUNT(*) FILTER (WHERE status = 'pending') as pending_orders
        FROM orders
        WHERE payment_status = 'paid'
        "#
    )
    .fetch_one(pool.get_ref())
    .await?;

    // User stats
    let users = sqlx::query!(
        r#"
        SELECT
            COUNT(*) as total_users,
            COUNT(*) FILTER (WHERE created_at >= date_trunc('month', NOW())) as new_this_month
        FROM users
        "#
    )
    .fetch_one(pool.get_ref())
    .await?;

    // Top products
    let top_products = sqlx::query!(
        r#"
        SELECT p.id, p.name, p.sku,
               SUM(oi.quantity) as total_sold,
               SUM(oi.total_price) as total_revenue
        FROM order_items oi
        JOIN products p ON oi.product_id = p.id
        JOIN orders o ON oi.order_id = o.id
        WHERE o.payment_status = 'paid'
        GROUP BY p.id, p.name, p.sku
        ORDER BY total_sold DESC
        LIMIT 10
        "#
    )
    .fetch_all(pool.get_ref())
    .await?;

    // Low stock alerts
    let low_stock = InventoryService::get_low_stock_products(pool.get_ref()).await?;

    Ok(HttpResponse::Ok().json(serde_json::json!({
        "revenue": {
            "today": revenue.today_revenue,
            "this_month": revenue.month_revenue,
            "total": revenue.total_revenue,
        },
        "orders": {
            "today": revenue.today_orders,
            "pending": revenue.pending_orders,
        },
        "users": {
            "total": users.total_users,
            "new_this_month": users.new_this_month,
        },
        "top_products": top_products,
        "low_stock_alerts": low_stock,
    })))
}
```

---

## 6. Main Application (`src/main.rs`)

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
    let stripe_key = env::var("STRIPE_API_KEY").unwrap_or_else(|_| "mock_key".to_string());
    let host = env::var("HOST").unwrap_or_else(|_| "127.0.0.1".to_string());
    let port = env::var("PORT").unwrap_or_else(|_| "8080".to_string());

    let pool = PgPoolOptions::new()
        .max_connections(20)
        .connect(&database_url)
        .await
        .expect("Failed to create pool");

    let stripe = web::Data::new(services::payment::MockStripeService::new(stripe_key));

    log::info!("Starting E-commerce API at http://{}:{}", host, port);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .wrap(Cors::permissive())
            .app_data(web::Data::new(pool.clone()))
            .app_data(stripe.clone())
            // Products
            .service(
                web::scope("/api/products")
                    .route("", web::get().to(handlers::products::list_products))
                    .route("", web::post().to(handlers::products::create_product))
                    .route("/{id}/stock", web::patch().to(handlers::products::update_stock))
            )
            // Categories
            .service(
                web::scope("/api/categories")
                    .route("", web::get().to(handlers::categories::list_categories))
                    .route("", web::post().to(handlers::categories::create_category))
            )
            // Cart
            .service(
                web::scope("/api/cart")
                    .route("", web::get().to(handlers::cart::get_cart))
                    .route("/items", web::post().to(handlers::cart::add_to_cart))
                    .route("/items/{id}", web::put().to(handlers::cart::update_cart_item))
                    .route("/items/{id}", web::delete().to(handlers::cart::remove_from_cart))
                    .route("/discount", web::post().to(handlers::cart::apply_discount))
                    .route("/shipping", web::get().to(handlers::cart::get_shipping_rates))
            )
            // Orders
            .service(
                web::scope("/api/orders")
                    .route("", web::post().to(handlers::orders::create_order))
                    .route("", web::get().to(handlers::orders::list_orders))
                    .route("/{id}", web::get().to(handlers::orders::get_order))
            )
            // Payments
            .service(
                web::scope("/api/payments")
                    .route("/confirm", web::post().to(handlers::payments::confirm_payment))
                    .route("/{id}/refund", web::post().to(handlers::payments::refund_payment))
            )
            // Admin
            .service(
                web::scope("/api/admin")
                    .route("/dashboard", web::get().to(handlers::admin::dashboard_stats))
                    .route("/orders", web::get().to(handlers::admin::list_all_orders))
                    .route("/orders/{id}/status", web::put().to(handlers::admin::update_order_status))
            )
    })
    .bind(format!("{}:{}", host, port))?
    .run()
    .await
}
```

---

## 7. Database Migrations

```sql
-- products table
CREATE TYPE product_status AS ENUM ('active', 'inactive', 'out_of_stock');

CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    slug VARCHAR(110) UNIQUE NOT NULL,
    description TEXT,
    parent_id UUID REFERENCES categories(id),
    image_url TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(220) UNIQUE NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL,
    compare_price NUMERIC(10, 2),
    cost_price NUMERIC(10, 2),
    sku VARCHAR(50) UNIQUE NOT NULL,
    barcode VARCHAR(50),
    category_id UUID REFERENCES categories(id),
    stock_quantity INTEGER NOT NULL DEFAULT 0,
    low_stock_threshold INTEGER NOT NULL DEFAULT 5,
    weight NUMERIC(8, 3),
    images TEXT[] NOT NULL DEFAULT '{}',
    tags TEXT[] NOT NULL DEFAULT '{}',
    status product_status NOT NULL DEFAULT 'active',
    is_featured BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- cart and order tables
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    price_at_time NUMERIC(10, 2) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(user_id, product_id)
);

CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded');
CREATE TYPE payment_status AS ENUM ('pending', 'paid', 'failed', 'refunded');

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(20) UNIQUE NOT NULL,
    user_id UUID NOT NULL REFERENCES users(id),
    status order_status NOT NULL DEFAULT 'pending',
    payment_status payment_status NOT NULL DEFAULT 'pending',
    payment_method VARCHAR(50),
    payment_intent_id VARCHAR(100),
    subtotal NUMERIC(10, 2) NOT NULL,
    discount_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
    shipping_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
    tax_amount NUMERIC(10, 2) NOT NULL DEFAULT 0,
    total NUMERIC(10, 2) NOT NULL,
    discount_code VARCHAR(50),
    shipping_address JSONB NOT NULL,
    billing_address JSONB,
    notes TEXT,
    shipped_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TYPE discount_type AS ENUM ('percentage', 'fixed_amount');

CREATE TABLE discount_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code VARCHAR(50) UNIQUE NOT NULL,
    discount_type discount_type NOT NULL,
    discount_value NUMERIC(10, 2) NOT NULL,
    minimum_order_amount NUMERIC(10, 2),
    maximum_uses INTEGER,
    current_uses INTEGER NOT NULL DEFAULT 0,
    expires_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## สรุป Part 082

ใน Part นี้เราได้สร้าง E-commerce API ที่สมบูรณ์ด้วย:
1. **Products & Categories** พร้อม inventory management
2. **Shopping Cart** พร้อม discount codes
3. **Order Processing** workflow ที่สมบูรณ์
4. **Mock Stripe Payment** integration
5. **Shipping Calculation** ตาม country และ weight
6. **Admin Dashboard** พร้อม stats และ low stock alerts

ใน **Part 083** เราจะสร้าง **Real-time Chat Backend** ด้วย WebSocket

---

*[← Part 081: Blog API](../part_081/README.md) | [Part 083: Real-time Chat Backend →](../part_083/README.md)*
