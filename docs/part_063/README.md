# Part 063: CQRS Pattern in Rust

## ภาพรวม

CQRS (Command Query Responsibility Segregation) คือ pattern ที่แยก operation ที่เปลี่ยนสถานะ (Commands) ออกจาก operation ที่อ่านข้อมูล (Queries) บทนี้จะสร้างระบบ Order Management ด้วย CQRS อย่างสมบูรณ์

## แนวคิดหลัก

```
Write Side (Commands)          Read Side (Queries)
┌──────────────────┐          ┌──────────────────┐
│   Command Bus    │          │   Query Bus      │
├──────────────────┤          ├──────────────────┤
│ Command Handlers │          │ Query Handlers   │
├──────────────────┤          ├──────────────────┤
│  Write Model     │ ──sync──▶│  Read Model      │
│  (Normalized DB) │          │  (Denormalized)  │
└──────────────────┘          └──────────────────┘
```

## Cargo.toml

```toml
[package]
name = "cqrs-orders"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
thiserror = "1"
async-trait = "0.1"
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio", "uuid", "chrono"] }
rust_decimal = { version = "1", features = ["serde-with-str"] }
```

## Command Definitions

```rust
// src/commands/mod.rs
use uuid::Uuid;
use serde::{Deserialize, Serialize};

/// Marker trait สำหรับ Commands
pub trait Command: Send + Sync + std::fmt::Debug {}

// Create Order Command
#[derive(Debug, Deserialize)]
pub struct CreateOrderCommand {
    pub customer_id: Uuid,
    pub customer_name: String,
    pub customer_email: String,
    pub items: Vec<CreateOrderItemCommand>,
    pub shipping_address: ShippingAddressCommand,
}

impl Command for CreateOrderCommand {}

#[derive(Debug, Deserialize)]
pub struct CreateOrderItemCommand {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: f64,
}

#[derive(Debug, Deserialize)]
pub struct ShippingAddressCommand {
    pub street: String,
    pub city: String,
    pub country: String,
    pub postal_code: String,
}

// Add Item Command
#[derive(Debug, Deserialize)]
pub struct AddItemToOrderCommand {
    pub order_id: Uuid,
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: f64,
}

impl Command for AddItemToOrderCommand {}

// Confirm Order Command
#[derive(Debug, Deserialize)]
pub struct ConfirmOrderCommand {
    pub order_id: Uuid,
    pub confirmed_by: String,
}

impl Command for ConfirmOrderCommand {}

// Cancel Order Command
#[derive(Debug, Deserialize)]
pub struct CancelOrderCommand {
    pub order_id: Uuid,
    pub reason: String,
    pub cancelled_by: String,
}

impl Command for CancelOrderCommand {}

// Ship Order Command
#[derive(Debug, Deserialize)]
pub struct ShipOrderCommand {
    pub order_id: Uuid,
    pub tracking_number: String,
    pub carrier: String,
}

impl Command for ShipOrderCommand {}
```

## Command Results

```rust
// src/commands/results.rs
use uuid::Uuid;
use serde::Serialize;

#[derive(Debug, Serialize)]
pub struct CreateOrderResult {
    pub order_id: Uuid,
    pub status: String,
}

#[derive(Debug, Serialize)]
pub struct CommandResult {
    pub success: bool,
    pub message: String,
}

impl CommandResult {
    pub fn success(message: impl Into<String>) -> Self {
        CommandResult { success: true, message: message.into() }
    }
    
    pub fn failure(message: impl Into<String>) -> Self {
        CommandResult { success: false, message: message.into() }
    }
}
```

## Write Model - Order Aggregate

```rust
// src/write_model/order.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use thiserror::Error;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
pub enum OrderStatus {
    Pending,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled,
}

impl std::fmt::Display for OrderStatus {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            OrderStatus::Pending => write!(f, "PENDING"),
            OrderStatus::Confirmed => write!(f, "CONFIRMED"),
            OrderStatus::Shipped => write!(f, "SHIPPED"),
            OrderStatus::Delivered => write!(f, "DELIVERED"),
            OrderStatus::Cancelled => write!(f, "CANCELLED"),
        }
    }
}

#[derive(Debug, Clone)]
pub struct OrderItem {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: f64,
}

impl OrderItem {
    pub fn subtotal(&self) -> f64 {
        self.unit_price * self.quantity as f64
    }
}

#[derive(Debug, Clone)]
pub struct ShippingAddress {
    pub street: String,
    pub city: String,
    pub country: String,
    pub postal_code: String,
}

#[derive(Debug, Error)]
pub enum OrderError {
    #[error("Cannot modify order in status: {0}")]
    InvalidStatus(String),
    #[error("Order is empty")]
    EmptyOrder,
    #[error("Product not found: {0}")]
    ProductNotFound(Uuid),
    #[error("Insufficient quantity for product: {0}")]
    InsufficientQuantity(Uuid),
}

/// Write Model - เน้น business rules และ invariants
pub struct OrderWriteModel {
    pub id: Uuid,
    pub customer_id: Uuid,
    pub status: OrderStatus,
    pub items: Vec<OrderItem>,
    pub shipping_address: Option<ShippingAddress>,
    pub version: u64, // สำหรับ optimistic locking
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

impl OrderWriteModel {
    pub fn new(customer_id: Uuid, shipping_address: ShippingAddress) -> Self {
        let now = Utc::now();
        OrderWriteModel {
            id: Uuid::new_v4(),
            customer_id,
            status: OrderStatus::Pending,
            items: Vec::new(),
            shipping_address: Some(shipping_address),
            version: 0,
            created_at: now,
            updated_at: now,
        }
    }
    
    pub fn add_item(
        &mut self,
        product_id: Uuid,
        product_name: String,
        quantity: u32,
        unit_price: f64,
    ) -> Result<(), OrderError> {
        if self.status != OrderStatus::Pending {
            return Err(OrderError::InvalidStatus(self.status.to_string()));
        }
        
        // รวม quantity ถ้า product มีอยู่แล้ว
        if let Some(item) = self.items.iter_mut().find(|i| i.product_id == product_id) {
            item.quantity += quantity;
        } else {
            self.items.push(OrderItem { product_id, product_name, quantity, unit_price });
        }
        
        self.updated_at = Utc::now();
        self.version += 1;
        Ok(())
    }
    
    pub fn confirm(&mut self) -> Result<(), OrderError> {
        if self.status != OrderStatus::Pending {
            return Err(OrderError::InvalidStatus(self.status.to_string()));
        }
        if self.items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }
        
        self.status = OrderStatus::Confirmed;
        self.updated_at = Utc::now();
        self.version += 1;
        Ok(())
    }
    
    pub fn ship(&mut self, tracking_number: String) -> Result<(), OrderError> {
        if self.status != OrderStatus::Confirmed {
            return Err(OrderError::InvalidStatus(self.status.to_string()));
        }
        
        self.status = OrderStatus::Shipped;
        self.updated_at = Utc::now();
        self.version += 1;
        Ok(())
    }
    
    pub fn cancel(&mut self, _reason: &str) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Pending | OrderStatus::Confirmed => {
                self.status = OrderStatus::Cancelled;
                self.updated_at = Utc::now();
                self.version += 1;
                Ok(())
            }
            _ => Err(OrderError::InvalidStatus(self.status.to_string())),
        }
    }
    
    pub fn total(&self) -> f64 {
        self.items.iter().map(|i| i.subtotal()).sum()
    }
}
```

## Command Handlers

```rust
// src/command_handlers/create_order_handler.rs
use async_trait::async_trait;
use std::sync::Arc;
use uuid::Uuid;

use crate::{
    commands::{CreateOrderCommand, Command},
    commands::results::CreateOrderResult,
    write_model::order::{OrderWriteModel, ShippingAddress},
    repositories::write::{OrderWriteRepository},
    events::{OrderCreatedEvent, EventPublisher},
};

pub struct CreateOrderHandler {
    write_repo: Arc<dyn OrderWriteRepository>,
    event_publisher: Arc<dyn EventPublisher>,
}

impl CreateOrderHandler {
    pub fn new(
        write_repo: Arc<dyn OrderWriteRepository>,
        event_publisher: Arc<dyn EventPublisher>,
    ) -> Self {
        CreateOrderHandler { write_repo, event_publisher }
    }
    
    pub async fn handle(&self, cmd: CreateOrderCommand) -> Result<CreateOrderResult, String> {
        // สร้าง write model
        let shipping = ShippingAddress {
            street: cmd.shipping_address.street,
            city: cmd.shipping_address.city,
            country: cmd.shipping_address.country,
            postal_code: cmd.shipping_address.postal_code,
        };
        
        let mut order = OrderWriteModel::new(cmd.customer_id, shipping);
        
        // เพิ่ม items
        for item in cmd.items {
            order.add_item(
                item.product_id,
                item.product_name,
                item.quantity,
                item.unit_price,
            ).map_err(|e| e.to_string())?;
        }
        
        let order_id = order.id;
        
        // บันทึก write model
        self.write_repo.save(&order).await.map_err(|e| e.to_string())?;
        
        // Publish event เพื่อ update read model
        self.event_publisher.publish(OrderCreatedEvent {
            order_id,
            customer_id: cmd.customer_id,
            customer_name: cmd.customer_name,
            customer_email: cmd.customer_email,
            total_amount: order.total(),
            item_count: order.items.len() as u32,
            status: order.status.to_string(),
            created_at: order.created_at,
        }).await.map_err(|e| e.to_string())?;
        
        Ok(CreateOrderResult {
            order_id,
            status: order.status.to_string(),
        })
    }
}

// src/command_handlers/confirm_order_handler.rs
use crate::commands::ConfirmOrderCommand;
use crate::commands::results::CommandResult;
use crate::events::OrderConfirmedEvent;

pub struct ConfirmOrderHandler {
    write_repo: Arc<dyn OrderWriteRepository>,
    event_publisher: Arc<dyn EventPublisher>,
}

impl ConfirmOrderHandler {
    pub fn new(
        write_repo: Arc<dyn OrderWriteRepository>,
        event_publisher: Arc<dyn EventPublisher>,
    ) -> Self {
        ConfirmOrderHandler { write_repo, event_publisher }
    }
    
    pub async fn handle(&self, cmd: ConfirmOrderCommand) -> Result<CommandResult, String> {
        let mut order = self.write_repo
            .find_by_id(cmd.order_id)
            .await
            .map_err(|e| e.to_string())?
            .ok_or_else(|| format!("Order {} not found", cmd.order_id))?;
        
        order.confirm().map_err(|e| e.to_string())?;
        
        self.write_repo.update(&order).await.map_err(|e| e.to_string())?;
        
        self.event_publisher.publish(OrderConfirmedEvent {
            order_id: cmd.order_id,
            confirmed_by: cmd.confirmed_by,
            confirmed_at: chrono::Utc::now(),
        }).await.map_err(|e| e.to_string())?;
        
        Ok(CommandResult::success(format!("Order {} confirmed", cmd.order_id)))
    }
}
```

## Events (สำหรับ sync Read Model)

```rust
// src/events/mod.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use async_trait::async_trait;
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCreatedEvent {
    pub order_id: Uuid,
    pub customer_id: Uuid,
    pub customer_name: String,
    pub customer_email: String,
    pub total_amount: f64,
    pub item_count: u32,
    pub status: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderConfirmedEvent {
    pub order_id: Uuid,
    pub confirmed_by: String,
    pub confirmed_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelledEvent {
    pub order_id: Uuid,
    pub reason: String,
    pub cancelled_at: DateTime<Utc>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderShippedEvent {
    pub order_id: Uuid,
    pub tracking_number: String,
    pub carrier: String,
    pub shipped_at: DateTime<Utc>,
}

// Event Publisher Trait
#[async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish<E: Serialize + Send + Sync>(&self, event: E) -> Result<(), String>;
}

// In-memory implementation สำหรับ testing
pub struct InMemoryEventPublisher;

#[async_trait]
impl EventPublisher for InMemoryEventPublisher {
    async fn publish<E: Serialize + Send + Sync>(&self, event: E) -> Result<(), String> {
        let json = serde_json::to_string(&event).map_err(|e| e.to_string())?;
        tracing::info!("Event published: {}", json);
        Ok(())
    }
}
```

## Read Models

```rust
// src/read_models/order_read_model.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// Read Model ออกแบบให้ query ได้ง่าย (denormalized)
#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct OrderSummaryReadModel {
    pub id: Uuid,
    pub customer_id: Uuid,
    pub customer_name: String,
    pub customer_email: String,
    pub status: String,
    pub total_amount: f64,
    pub item_count: i32,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

/// Read Model สำหรับรายละเอียด order
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderDetailReadModel {
    pub id: Uuid,
    pub customer_id: Uuid,
    pub customer_name: String,
    pub customer_email: String,
    pub status: String,
    pub items: Vec<OrderItemReadModel>,
    pub shipping_address: ShippingAddressReadModel,
    pub total_amount: f64,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
    pub tracking_number: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize, sqlx::FromRow)]
pub struct OrderItemReadModel {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: i32,
    pub unit_price: f64,
    pub subtotal: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ShippingAddressReadModel {
    pub street: String,
    pub city: String,
    pub country: String,
    pub postal_code: String,
}
```

## Query Definitions

```rust
// src/queries/mod.rs
use uuid::Uuid;
use serde::Deserialize;

pub trait Query: Send + Sync + std::fmt::Debug {}

// Get Order by ID
#[derive(Debug, Deserialize)]
pub struct GetOrderByIdQuery {
    pub order_id: Uuid,
}
impl Query for GetOrderByIdQuery {}

// List Orders
#[derive(Debug, Deserialize)]
pub struct ListOrdersQuery {
    pub customer_id: Option<Uuid>,
    pub status: Option<String>,
    pub page: u32,
    pub per_page: u32,
}
impl Query for ListOrdersQuery {}

// Get Orders by Status
#[derive(Debug, Deserialize)]
pub struct GetOrdersByStatusQuery {
    pub status: String,
    pub limit: u32,
}
impl Query for GetOrdersByStatusQuery {}

// Search Orders
#[derive(Debug, Deserialize)]
pub struct SearchOrdersQuery {
    pub search_term: String,
    pub page: u32,
    pub per_page: u32,
}
impl Query for SearchOrdersQuery {}
```

## Query Handlers

```rust
// src/query_handlers/get_order_handler.rs
use async_trait::async_trait;
use std::sync::Arc;

use crate::{
    queries::GetOrderByIdQuery,
    read_models::OrderDetailReadModel,
    repositories::read::OrderReadRepository,
};

pub struct GetOrderByIdHandler {
    read_repo: Arc<dyn OrderReadRepository>,
}

impl GetOrderByIdHandler {
    pub fn new(read_repo: Arc<dyn OrderReadRepository>) -> Self {
        GetOrderByIdHandler { read_repo }
    }
    
    pub async fn handle(&self, query: GetOrderByIdQuery) -> Result<Option<OrderDetailReadModel>, String> {
        self.read_repo
            .find_by_id(query.order_id)
            .await
            .map_err(|e| e.to_string())
    }
}

// src/query_handlers/list_orders_handler.rs
use crate::{
    queries::ListOrdersQuery,
    read_models::OrderSummaryReadModel,
};

pub struct ListOrdersResult {
    pub orders: Vec<OrderSummaryReadModel>,
    pub total: u64,
    pub page: u32,
    pub per_page: u32,
}

pub struct ListOrdersHandler {
    read_repo: Arc<dyn OrderReadRepository>,
}

impl ListOrdersHandler {
    pub fn new(read_repo: Arc<dyn OrderReadRepository>) -> Self {
        ListOrdersHandler { read_repo }
    }
    
    pub async fn handle(&self, query: ListOrdersQuery) -> Result<ListOrdersResult, String> {
        let (orders, total) = self.read_repo
            .find_all(query.customer_id, query.status, query.page, query.per_page)
            .await
            .map_err(|e| e.to_string())?;
        
        Ok(ListOrdersResult {
            orders,
            total,
            page: query.page,
            per_page: query.per_page,
        })
    }
}
```

## Repository Traits

```rust
// src/repositories/write.rs
use async_trait::async_trait;
use uuid::Uuid;
use crate::write_model::order::OrderWriteModel;

#[async_trait]
pub trait OrderWriteRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<OrderWriteModel>, Box<dyn std::error::Error>>;
    async fn save(&self, order: &OrderWriteModel) -> Result<(), Box<dyn std::error::Error>>;
    async fn update(&self, order: &OrderWriteModel) -> Result<(), Box<dyn std::error::Error>>;
    async fn delete(&self, id: Uuid) -> Result<(), Box<dyn std::error::Error>>;
}

// src/repositories/read.rs
use async_trait::async_trait;
use uuid::Uuid;
use crate::read_models::{OrderDetailReadModel, OrderSummaryReadModel};

#[async_trait]
pub trait OrderReadRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<OrderDetailReadModel>, Box<dyn std::error::Error>>;
    async fn find_all(
        &self,
        customer_id: Option<Uuid>,
        status: Option<String>,
        page: u32,
        per_page: u32,
    ) -> Result<(Vec<OrderSummaryReadModel>, u64), Box<dyn std::error::Error>>;
    async fn update_from_event(&self, event: serde_json::Value) -> Result<(), Box<dyn std::error::Error>>;
}
```

## Projections (Read Model Updaters)

```rust
// src/projections/order_projection.rs
use async_trait::async_trait;
use std::sync::Arc;
use sqlx::PgPool;

use crate::events::{OrderCreatedEvent, OrderConfirmedEvent, OrderShippedEvent, OrderCancelledEvent};
use crate::repositories::read::OrderReadRepository;

/// Projection อัพเดท read model เมื่อมี event ใหม่
pub struct OrderProjection {
    pool: PgPool,
}

impl OrderProjection {
    pub fn new(pool: PgPool) -> Self {
        OrderProjection { pool }
    }
    
    pub async fn on_order_created(&self, event: OrderCreatedEvent) -> Result<(), sqlx::Error> {
        sqlx::query!(
            r#"
            INSERT INTO order_summaries (
                id, customer_id, customer_name, customer_email,
                status, total_amount, item_count, created_at, updated_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $8)
            ON CONFLICT (id) DO NOTHING
            "#,
            event.order_id,
            event.customer_id,
            event.customer_name,
            event.customer_email,
            event.status,
            event.total_amount,
            event.item_count as i32,
            event.created_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn on_order_confirmed(&self, event: OrderConfirmedEvent) -> Result<(), sqlx::Error> {
        sqlx::query!(
            r#"
            UPDATE order_summaries
            SET status = 'CONFIRMED', updated_at = $2
            WHERE id = $1
            "#,
            event.order_id,
            event.confirmed_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn on_order_shipped(&self, event: OrderShippedEvent) -> Result<(), sqlx::Error> {
        sqlx::query!(
            r#"
            UPDATE order_summaries
            SET status = 'SHIPPED', updated_at = $2
            WHERE id = $1
            "#,
            event.order_id,
            event.shipped_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn on_order_cancelled(&self, event: OrderCancelledEvent) -> Result<(), sqlx::Error> {
        sqlx::query!(
            r#"
            UPDATE order_summaries
            SET status = 'CANCELLED', updated_at = $2
            WHERE id = $1
            "#,
            event.order_id,
            event.cancelled_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
}
```

## Actix-web HTTP Layer

```rust
// src/interface/http/order_controller.rs
use actix_web::{web, HttpResponse};
use std::sync::Arc;
use uuid::Uuid;
use serde::Deserialize;

use crate::{
    commands::{CreateOrderCommand, ConfirmOrderCommand, CancelOrderCommand, ShipOrderCommand},
    command_handlers::{CreateOrderHandler, ConfirmOrderHandler, CancelOrderHandler, ShipOrderHandler},
    queries::{GetOrderByIdQuery, ListOrdersQuery},
    query_handlers::{GetOrderByIdHandler, ListOrdersHandler},
};

pub struct OrderControllerState {
    // Command handlers
    pub create_handler: Arc<CreateOrderHandler>,
    pub confirm_handler: Arc<ConfirmOrderHandler>,
    pub cancel_handler: Arc<CancelOrderHandler>,
    pub ship_handler: Arc<ShipOrderHandler>,
    
    // Query handlers
    pub get_order_handler: Arc<GetOrderByIdHandler>,
    pub list_orders_handler: Arc<ListOrdersHandler>,
}

// Command endpoints
pub async fn create_order(
    state: web::Data<OrderControllerState>,
    body: web::Json<CreateOrderCommand>,
) -> HttpResponse {
    match state.create_handler.handle(body.into_inner()).await {
        Ok(result) => HttpResponse::Created().json(result),
        Err(e) => HttpResponse::BadRequest().json(serde_json::json!({"error": e})),
    }
}

pub async fn confirm_order(
    state: web::Data<OrderControllerState>,
    path: web::Path<Uuid>,
    body: web::Json<ConfirmOrderBody>,
) -> HttpResponse {
    let cmd = ConfirmOrderCommand {
        order_id: *path,
        confirmed_by: body.confirmed_by.clone(),
    };
    
    match state.confirm_handler.handle(cmd).await {
        Ok(result) => HttpResponse::Ok().json(result),
        Err(e) => HttpResponse::BadRequest().json(serde_json::json!({"error": e})),
    }
}

pub async fn cancel_order(
    state: web::Data<OrderControllerState>,
    path: web::Path<Uuid>,
    body: web::Json<CancelOrderBody>,
) -> HttpResponse {
    let cmd = CancelOrderCommand {
        order_id: *path,
        reason: body.reason.clone(),
        cancelled_by: body.cancelled_by.clone(),
    };
    
    match state.cancel_handler.handle(cmd).await {
        Ok(result) => HttpResponse::Ok().json(result),
        Err(e) => HttpResponse::BadRequest().json(serde_json::json!({"error": e})),
    }
}

// Query endpoints
pub async fn get_order(
    state: web::Data<OrderControllerState>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    let query = GetOrderByIdQuery { order_id: *path };
    
    match state.get_order_handler.handle(query).await {
        Ok(Some(order)) => HttpResponse::Ok().json(order),
        Ok(None) => HttpResponse::NotFound().json(serde_json::json!({"error": "Order not found"})),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

#[derive(Deserialize)]
pub struct ListOrdersParams {
    pub customer_id: Option<Uuid>,
    pub status: Option<String>,
    pub page: Option<u32>,
    pub per_page: Option<u32>,
}

pub async fn list_orders(
    state: web::Data<OrderControllerState>,
    params: web::Query<ListOrdersParams>,
) -> HttpResponse {
    let query = ListOrdersQuery {
        customer_id: params.customer_id,
        status: params.status.clone(),
        page: params.page.unwrap_or(1),
        per_page: params.per_page.unwrap_or(20),
    };
    
    match state.list_orders_handler.handle(query).await {
        Ok(result) => HttpResponse::Ok().json(result),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

#[derive(Deserialize)]
pub struct ConfirmOrderBody {
    pub confirmed_by: String,
}

#[derive(Deserialize)]
pub struct CancelOrderBody {
    pub reason: String,
    pub cancelled_by: String,
}

// Router
pub fn order_routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/orders")
            // Commands (write)
            .route("", web::post().to(create_order))
            .route("/{id}/confirm", web::post().to(confirm_order))
            .route("/{id}/cancel", web::post().to(cancel_order))
            // Queries (read)
            .route("", web::get().to(list_orders))
            .route("/{id}", web::get().to(get_order))
    );
}
```

## Eventual Consistency Example

```rust
// src/sync/read_model_sync.rs
use tokio::sync::mpsc;
use std::sync::Arc;

use crate::{
    events::{OrderCreatedEvent, OrderConfirmedEvent},
    projections::OrderProjection,
};

pub enum SyncEvent {
    OrderCreated(OrderCreatedEvent),
    OrderConfirmed(OrderConfirmedEvent),
    // ... events อื่น ๆ
}

/// Background task สำหรับ sync read model
/// ใช้ channel เพื่อรับ events
pub struct ReadModelSyncer {
    projection: Arc<OrderProjection>,
    receiver: mpsc::Receiver<SyncEvent>,
}

impl ReadModelSyncer {
    pub fn new(
        projection: Arc<OrderProjection>,
        receiver: mpsc::Receiver<SyncEvent>,
    ) -> Self {
        ReadModelSyncer { projection, receiver }
    }
    
    pub async fn run(mut self) {
        while let Some(event) = self.receiver.recv().await {
            let result = match event {
                SyncEvent::OrderCreated(e) => {
                    self.projection.on_order_created(e).await.map_err(|e| e.to_string())
                }
                SyncEvent::OrderConfirmed(e) => {
                    self.projection.on_order_confirmed(e).await.map_err(|e| e.to_string())
                }
            };
            
            if let Err(e) = result {
                tracing::error!("Failed to sync read model: {}", e);
                // ใน production ควร retry หรือ dead letter queue
            }
        }
    }
}

// Channel-based event publisher
pub struct ChannelEventPublisher {
    sender: mpsc::Sender<SyncEvent>,
}

impl ChannelEventPublisher {
    pub fn new(sender: mpsc::Sender<SyncEvent>) -> Self {
        ChannelEventPublisher { sender }
    }
    
    pub async fn publish_order_created(&self, event: OrderCreatedEvent) -> Result<(), String> {
        self.sender
            .send(SyncEvent::OrderCreated(event))
            .await
            .map_err(|e| e.to_string())
    }
}
```

## Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use uuid::Uuid;
    
    #[test]
    fn test_create_order_write_model() {
        let customer_id = Uuid::new_v4();
        let shipping = ShippingAddress {
            street: "123 Test St".to_string(),
            city: "Bangkok".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        };
        
        let order = OrderWriteModel::new(customer_id, shipping);
        assert_eq!(order.customer_id, customer_id);
        assert_eq!(order.status, OrderStatus::Pending);
        assert!(order.items.is_empty());
    }
    
    #[test]
    fn test_add_item_and_confirm() {
        let customer_id = Uuid::new_v4();
        let shipping = ShippingAddress {
            street: "123 Test St".to_string(),
            city: "Bangkok".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        };
        
        let mut order = OrderWriteModel::new(customer_id, shipping);
        
        order.add_item(Uuid::new_v4(), "Product A".to_string(), 2, 100.0).unwrap();
        order.add_item(Uuid::new_v4(), "Product B".to_string(), 1, 200.0).unwrap();
        
        assert_eq!(order.total(), 400.0); // 2*100 + 1*200
        
        order.confirm().unwrap();
        assert_eq!(order.status, OrderStatus::Confirmed);
    }
    
    #[test]
    fn test_cannot_add_item_after_confirm() {
        let customer_id = Uuid::new_v4();
        let shipping = ShippingAddress {
            street: "Test".to_string(),
            city: "Bangkok".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        };
        
        let mut order = OrderWriteModel::new(customer_id, shipping);
        order.add_item(Uuid::new_v4(), "Product".to_string(), 1, 100.0).unwrap();
        order.confirm().unwrap();
        
        let result = order.add_item(Uuid::new_v4(), "More".to_string(), 1, 50.0);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_cancel_after_confirm() {
        let customer_id = Uuid::new_v4();
        let shipping = ShippingAddress {
            street: "Test".to_string(),
            city: "Bangkok".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        };
        
        let mut order = OrderWriteModel::new(customer_id, shipping);
        order.add_item(Uuid::new_v4(), "Product".to_string(), 1, 100.0).unwrap();
        order.confirm().unwrap();
        order.cancel("Customer request").unwrap();
        
        assert_eq!(order.status, OrderStatus::Cancelled);
    }
    
    #[test]
    fn test_version_increments() {
        let customer_id = Uuid::new_v4();
        let shipping = ShippingAddress {
            street: "Test".to_string(),
            city: "Bangkok".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        };
        
        let mut order = OrderWriteModel::new(customer_id, shipping);
        assert_eq!(order.version, 0);
        
        order.add_item(Uuid::new_v4(), "Product".to_string(), 1, 100.0).unwrap();
        assert_eq!(order.version, 1);
        
        order.confirm().unwrap();
        assert_eq!(order.version, 2);
    }
}
```

## สรุป

CQRS ใน Rust มีประโยชน์เมื่อ:

1. **Read/Write workloads ต่างกันมาก** - scale ได้อิสระ
2. **Complex queries** - read model ออกแบบเฉพาะ
3. **Audit trail** - เก็บ events ทุก state change
4. **Multiple read models** - สำหรับ use cases ต่างกัน

ข้อเสีย: Eventual consistency, complexity เพิ่มขึ้น

---

## Navigation

- [← Part 062: Domain-Driven Design](../part_062/README.md)
- [→ Part 064: Event Sourcing](../part_064/README.md)
- [กลับหน้าหลัก](../../README.md)
