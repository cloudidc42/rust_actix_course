# Part 062: Domain-Driven Design (DDD) in Rust

## ภาพรวม

Domain-Driven Design (DDD) เป็นแนวคิดการออกแบบซอฟต์แวร์ที่เน้นการสร้าง domain model ที่สะท้อนภาษาและกระบวนการทางธุรกิจอย่างชัดเจน บทนี้จะนำ DDD มาประยุกต์ใช้กับ Rust สำหรับระบบ e-commerce

## แนวคิดหลักของ DDD

```
DDD Tactical Patterns:
├── Entities          - มี identity, lifecycle
├── Value Objects     - ไม่มี identity, immutable
├── Aggregates        - cluster ของ entities/value objects
├── Aggregate Roots   - entry point ของ aggregate
├── Domain Services   - logic ที่ไม่เป็นของ entity ใด
├── Domain Events     - สิ่งที่เกิดขึ้นใน domain
├── Repositories      - persistence abstraction
└── Factories         - สร้าง complex objects
```

## Bounded Contexts

```
E-Commerce System:
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   Catalog BC     │  │   Order BC       │  │   Payment BC     │
│                  │  │                  │  │                  │
│ - Product        │  │ - Order          │  │ - Payment        │
│ - Category       │  │ - OrderItem      │  │ - Invoice        │
│ - Price          │  │ - ShippingAddr   │  │ - Transaction    │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

## Cargo.toml

```toml
[package]
name = "ddd-ecommerce"
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
rust_decimal = { version = "1", features = ["serde-with-str"] }
```

## Value Objects

```rust
// src/domain/value_objects/money.rs
use rust_decimal::Decimal;
use std::fmt;
use thiserror::Error;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Money {
    amount: Decimal,
    currency: Currency,
}

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub enum Currency {
    THB,
    USD,
    EUR,
}

#[derive(Debug, Error)]
pub enum MoneyError {
    #[error("Amount cannot be negative: {0}")]
    NegativeAmount(Decimal),
    #[error("Currency mismatch: cannot operate on {0:?} and {1:?}")]
    CurrencyMismatch(Currency, Currency),
}

impl Money {
    pub fn new(amount: Decimal, currency: Currency) -> Result<Self, MoneyError> {
        if amount < Decimal::ZERO {
            return Err(MoneyError::NegativeAmount(amount));
        }
        Ok(Money { amount, currency })
    }
    
    pub fn zero(currency: Currency) -> Self {
        Money {
            amount: Decimal::ZERO,
            currency,
        }
    }
    
    pub fn add(&self, other: &Money) -> Result<Money, MoneyError> {
        if self.currency != other.currency {
            return Err(MoneyError::CurrencyMismatch(
                self.currency.clone(),
                other.currency.clone(),
            ));
        }
        Ok(Money {
            amount: self.amount + other.amount,
            currency: self.currency.clone(),
        })
    }
    
    pub fn multiply(&self, quantity: u32) -> Money {
        Money {
            amount: self.amount * Decimal::from(quantity),
            currency: self.currency.clone(),
        }
    }
    
    pub fn amount(&self) -> Decimal { self.amount }
    pub fn currency(&self) -> &Currency { &self.currency }
}

impl fmt::Display for Money {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} {:?}", self.amount, self.currency)
    }
}

// src/domain/value_objects/quantity.rs
use thiserror::Error;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
pub struct Quantity(u32);

#[derive(Debug, Error)]
pub enum QuantityError {
    #[error("Quantity cannot be zero")]
    Zero,
    #[error("Quantity {0} exceeds maximum {1}")]
    ExceedsMaximum(u32, u32),
}

impl Quantity {
    pub fn new(value: u32) -> Result<Self, QuantityError> {
        if value == 0 {
            return Err(QuantityError::Zero);
        }
        Ok(Quantity(value))
    }
    
    pub fn new_with_max(value: u32, max: u32) -> Result<Self, QuantityError> {
        if value == 0 {
            return Err(QuantityError::Zero);
        }
        if value > max {
            return Err(QuantityError::ExceedsMaximum(value, max));
        }
        Ok(Quantity(value))
    }
    
    pub fn value(&self) -> u32 { self.0 }
}

// src/domain/value_objects/address.rs
use thiserror::Error;

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Address {
    street: String,
    city: String,
    postal_code: PostalCode,
    country: String,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub struct PostalCode(String);

#[derive(Debug, Error)]
pub enum AddressError {
    #[error("Invalid postal code: {0}")]
    InvalidPostalCode(String),
    #[error("Street cannot be empty")]
    EmptyStreet,
    #[error("City cannot be empty")]
    EmptyCity,
}

impl PostalCode {
    pub fn new(code: impl Into<String>) -> Result<Self, AddressError> {
        let code = code.into();
        // ตรวจสอบรหัสไปรษณีย์ไทย (5 หลัก)
        if code.len() == 5 && code.chars().all(|c| c.is_ascii_digit()) {
            Ok(PostalCode(code))
        } else {
            Err(AddressError::InvalidPostalCode(code))
        }
    }
}

impl Address {
    pub fn new(
        street: String,
        city: String,
        postal_code: PostalCode,
        country: String,
    ) -> Result<Self, AddressError> {
        if street.trim().is_empty() {
            return Err(AddressError::EmptyStreet);
        }
        if city.trim().is_empty() {
            return Err(AddressError::EmptyCity);
        }
        Ok(Address { street, city, postal_code, country })
    }
    
    pub fn street(&self) -> &str { &self.street }
    pub fn city(&self) -> &str { &self.city }
    pub fn postal_code(&self) -> &PostalCode { &self.postal_code }
    pub fn country(&self) -> &str { &self.country }
}
```

## Domain Events

```rust
// src/domain/events/mod.rs
use chrono::{DateTime, Utc};
use uuid::Uuid;
use serde::{Deserialize, Serialize};

// Base trait สำหรับ domain events ทุกตัว
pub trait DomainEvent: Send + Sync {
    fn event_id(&self) -> Uuid;
    fn occurred_at(&self) -> DateTime<Utc>;
    fn event_type(&self) -> &'static str;
}

// Order domain events
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderPlaced {
    pub event_id: Uuid,
    pub order_id: Uuid,
    pub customer_id: Uuid,
    pub total_amount: String,
    pub occurred_at: DateTime<Utc>,
}

impl DomainEvent for OrderPlaced {
    fn event_id(&self) -> Uuid { self.event_id }
    fn occurred_at(&self) -> DateTime<Utc> { self.occurred_at }
    fn event_type(&self) -> &'static str { "OrderPlaced" }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderConfirmed {
    pub event_id: Uuid,
    pub order_id: Uuid,
    pub confirmed_at: DateTime<Utc>,
    pub occurred_at: DateTime<Utc>,
}

impl DomainEvent for OrderConfirmed {
    fn event_id(&self) -> Uuid { self.event_id }
    fn occurred_at(&self) -> DateTime<Utc> { self.occurred_at }
    fn event_type(&self) -> &'static str { "OrderConfirmed" }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct OrderCancelled {
    pub event_id: Uuid,
    pub order_id: Uuid,
    pub reason: String,
    pub occurred_at: DateTime<Utc>,
}

impl DomainEvent for OrderCancelled {
    fn event_id(&self) -> Uuid { self.event_id }
    fn occurred_at(&self) -> DateTime<Utc> { self.occurred_at }
    fn event_type(&self) -> &'static str { "OrderCancelled" }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ItemAddedToOrder {
    pub event_id: Uuid,
    pub order_id: Uuid,
    pub product_id: Uuid,
    pub quantity: u32,
    pub unit_price: String,
    pub occurred_at: DateTime<Utc>,
}

impl DomainEvent for ItemAddedToOrder {
    fn event_id(&self) -> Uuid { self.event_id }
    fn occurred_at(&self) -> DateTime<Utc> { self.occurred_at }
    fn event_type(&self) -> &'static str { "ItemAddedToOrder" }
}

// Enum สำหรับ type-safe event handling
#[derive(Debug, Clone)]
pub enum OrderEvent {
    Placed(OrderPlaced),
    Confirmed(OrderConfirmed),
    Cancelled(OrderCancelled),
    ItemAdded(ItemAddedToOrder),
}
```

## Aggregate Root - Order

```rust
// src/domain/aggregates/order.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use thiserror::Error;

use crate::domain::{
    value_objects::{Money, Quantity, Address},
    events::{OrderEvent, OrderPlaced, OrderConfirmed, OrderCancelled, ItemAddedToOrder},
};

#[derive(Debug, Clone, PartialEq)]
pub enum OrderStatus {
    Draft,
    Placed,
    Confirmed,
    Shipped,
    Delivered,
    Cancelled,
}

#[derive(Debug, Clone)]
pub struct OrderItem {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: Quantity,
    pub unit_price: Money,
}

impl OrderItem {
    pub fn subtotal(&self) -> Money {
        self.unit_price.multiply(self.quantity.value())
    }
}

#[derive(Debug, Error)]
pub enum OrderError {
    #[error("Cannot add items to order in status: {0:?}")]
    InvalidStatusForAddItem(OrderStatus),
    #[error("Cannot place empty order")]
    EmptyOrder,
    #[error("Order already placed")]
    AlreadyPlaced,
    #[error("Cannot confirm order in status: {0:?}")]
    InvalidStatusForConfirm(OrderStatus),
    #[error("Cannot cancel order in status: {0:?}")]
    InvalidStatusForCancel(OrderStatus),
    #[error("Maximum items exceeded")]
    MaxItemsExceeded,
    #[error("Product {0} not found in order")]
    ProductNotFound(Uuid),
}

const MAX_ORDER_ITEMS: usize = 50;

/// Order เป็น Aggregate Root
/// ทุกการเปลี่ยนแปลงต้องผ่าน Order เท่านั้น
pub struct Order {
    id: Uuid,
    customer_id: Uuid,
    status: OrderStatus,
    items: Vec<OrderItem>,
    shipping_address: Option<Address>,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
    // เก็บ domain events ที่ยังไม่ได้ publish
    pending_events: Vec<OrderEvent>,
}

impl Order {
    pub fn new(customer_id: Uuid) -> Self {
        let now = Utc::now();
        Order {
            id: Uuid::new_v4(),
            customer_id,
            status: OrderStatus::Draft,
            items: Vec::new(),
            shipping_address: None,
            created_at: now,
            updated_at: now,
            pending_events: Vec::new(),
        }
    }
    
    /// เพิ่มสินค้าใน order (business rule)
    pub fn add_item(
        &mut self,
        product_id: Uuid,
        product_name: String,
        quantity: Quantity,
        unit_price: Money,
    ) -> Result<(), OrderError> {
        // Business rule: ต้องอยู่ใน Draft status
        if self.status != OrderStatus::Draft {
            return Err(OrderError::InvalidStatusForAddItem(self.status.clone()));
        }
        
        // Business rule: ไม่เกิน 50 items
        if self.items.len() >= MAX_ORDER_ITEMS {
            return Err(OrderError::MaxItemsExceeded);
        }
        
        // ถ้ามีสินค้านี้อยู่แล้ว ให้เพิ่ม quantity
        if let Some(existing) = self.items.iter_mut().find(|i| i.product_id == product_id) {
            let new_qty = Quantity::new(existing.quantity.value() + quantity.value())
                .map_err(|_| OrderError::MaxItemsExceeded)?;
            existing.quantity = new_qty;
        } else {
            self.items.push(OrderItem {
                product_id,
                product_name,
                quantity: quantity.clone(),
                unit_price: unit_price.clone(),
            });
        }
        
        // บันทึก domain event
        self.pending_events.push(OrderEvent::ItemAdded(ItemAddedToOrder {
            event_id: Uuid::new_v4(),
            order_id: self.id,
            product_id,
            quantity: quantity.value(),
            unit_price: unit_price.to_string(),
            occurred_at: Utc::now(),
        }));
        
        self.updated_at = Utc::now();
        Ok(())
    }
    
    /// Place order
    pub fn place(&mut self, shipping_address: Address) -> Result<(), OrderError> {
        if self.status != OrderStatus::Draft {
            return Err(OrderError::AlreadyPlaced);
        }
        
        if self.items.is_empty() {
            return Err(OrderError::EmptyOrder);
        }
        
        self.shipping_address = Some(shipping_address);
        self.status = OrderStatus::Placed;
        self.updated_at = Utc::now();
        
        // บันทึก domain event
        self.pending_events.push(OrderEvent::Placed(OrderPlaced {
            event_id: Uuid::new_v4(),
            order_id: self.id,
            customer_id: self.customer_id,
            total_amount: self.total().to_string(),
            occurred_at: Utc::now(),
        }));
        
        Ok(())
    }
    
    /// Confirm order
    pub fn confirm(&mut self) -> Result<(), OrderError> {
        if self.status != OrderStatus::Placed {
            return Err(OrderError::InvalidStatusForConfirm(self.status.clone()));
        }
        
        self.status = OrderStatus::Confirmed;
        self.updated_at = Utc::now();
        
        self.pending_events.push(OrderEvent::Confirmed(OrderConfirmed {
            event_id: Uuid::new_v4(),
            order_id: self.id,
            confirmed_at: Utc::now(),
            occurred_at: Utc::now(),
        }));
        
        Ok(())
    }
    
    /// Cancel order
    pub fn cancel(&mut self, reason: String) -> Result<(), OrderError> {
        match self.status {
            OrderStatus::Draft | OrderStatus::Placed | OrderStatus::Confirmed => {
                self.status = OrderStatus::Cancelled;
                self.updated_at = Utc::now();
                
                self.pending_events.push(OrderEvent::Cancelled(OrderCancelled {
                    event_id: Uuid::new_v4(),
                    order_id: self.id,
                    reason,
                    occurred_at: Utc::now(),
                }));
                
                Ok(())
            }
            _ => Err(OrderError::InvalidStatusForCancel(self.status.clone())),
        }
    }
    
    /// คำนวณ total
    pub fn total(&self) -> Money {
        if self.items.is_empty() {
            return Money::zero(crate::domain::value_objects::Currency::THB);
        }
        
        self.items.iter()
            .fold(
                Money::zero(self.items[0].unit_price.currency().clone()),
                |acc, item| acc.add(&item.subtotal()).unwrap()
            )
    }
    
    /// เอา pending events ออก (หลัง publish แล้ว)
    pub fn take_events(&mut self) -> Vec<OrderEvent> {
        std::mem::take(&mut self.pending_events)
    }
    
    // Getters
    pub fn id(&self) -> Uuid { self.id }
    pub fn customer_id(&self) -> Uuid { self.customer_id }
    pub fn status(&self) -> &OrderStatus { &self.status }
    pub fn items(&self) -> &[OrderItem] { &self.items }
    pub fn shipping_address(&self) -> Option<&Address> { self.shipping_address.as_ref() }
    pub fn created_at(&self) -> DateTime<Utc> { self.created_at }
    pub fn updated_at(&self) -> DateTime<Utc> { self.updated_at }
}
```

## Domain Services

```rust
// src/domain/services/pricing_service.rs
use rust_decimal::Decimal;
use crate::domain::value_objects::{Money, Currency, Quantity};

/// Domain Service สำหรับ logic ที่ไม่เป็นของ entity ใด
pub struct PricingService;

impl PricingService {
    /// คำนวณราคาหลังส่วนลด
    pub fn apply_discount(price: &Money, discount_percent: Decimal) -> Result<Money, String> {
        if discount_percent < Decimal::ZERO || discount_percent > Decimal::from(100) {
            return Err("Discount must be between 0 and 100".to_string());
        }
        
        let discount = price.amount() * (discount_percent / Decimal::from(100));
        let discounted = price.amount() - discount;
        
        Money::new(discounted, price.currency().clone())
            .map_err(|e| e.to_string())
    }
    
    /// คำนวณราคาสำหรับการซื้อ bulk
    pub fn bulk_price(unit_price: &Money, quantity: &Quantity) -> Money {
        let qty = Decimal::from(quantity.value());
        let total = unit_price.amount() * qty;
        
        // Business rule: ซื้อมากกว่า 10 ชิ้นได้ส่วนลด 10%
        let final_total = if quantity.value() >= 10 {
            total * Decimal::new(90, 2) // 0.90
        } else {
            total
        };
        
        Money::new(final_total, unit_price.currency().clone()).unwrap()
    }
    
    /// ตรวจสอบว่าราคาสมเหตุสมผล
    pub fn validate_price(price: &Money) -> bool {
        price.amount() > Decimal::ZERO && 
        price.amount() < Decimal::from(1_000_000)
    }
}

// src/domain/services/inventory_service.rs
use async_trait::async_trait;
use uuid::Uuid;

#[async_trait]
pub trait InventoryService: Send + Sync {
    async fn check_availability(&self, product_id: Uuid, quantity: u32) -> Result<bool, String>;
    async fn reserve(&self, product_id: Uuid, quantity: u32) -> Result<(), String>;
    async fn release(&self, product_id: Uuid, quantity: u32) -> Result<(), String>;
}
```

## Repository Interface

```rust
// src/domain/repositories/order_repository.rs
use async_trait::async_trait;
use uuid::Uuid;
use crate::domain::{
    aggregates::Order,
    errors::DomainError,
};

#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, DomainError>;
    async fn find_by_customer(&self, customer_id: Uuid) -> Result<Vec<Order>, DomainError>;
    async fn save(&self, order: &Order) -> Result<(), DomainError>;
    async fn update(&self, order: &Order) -> Result<(), DomainError>;
}
```

## Application Service (Use Case with Domain Events)

```rust
// src/application/order_service.rs
use std::sync::Arc;
use uuid::Uuid;

use crate::domain::{
    aggregates::Order,
    repositories::OrderRepository,
    services::InventoryService,
    value_objects::{Address, Money, Quantity, Currency, PostalCode},
    events::OrderEvent,
};

pub struct PlaceOrderCommand {
    pub customer_id: Uuid,
    pub items: Vec<OrderItemCommand>,
    pub shipping_street: String,
    pub shipping_city: String,
    pub shipping_postal_code: String,
    pub shipping_country: String,
}

pub struct OrderItemCommand {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price_amount: String,
}

pub struct OrderApplicationService {
    order_repository: Arc<dyn OrderRepository>,
    inventory_service: Arc<dyn InventoryService>,
    event_publisher: Arc<dyn EventPublisher>,
}

#[async_trait::async_trait]
pub trait EventPublisher: Send + Sync {
    async fn publish(&self, events: Vec<OrderEvent>) -> Result<(), String>;
}

impl OrderApplicationService {
    pub fn new(
        order_repository: Arc<dyn OrderRepository>,
        inventory_service: Arc<dyn InventoryService>,
        event_publisher: Arc<dyn EventPublisher>,
    ) -> Self {
        OrderApplicationService {
            order_repository,
            inventory_service,
            event_publisher,
        }
    }
    
    pub async fn place_order(&self, cmd: PlaceOrderCommand) -> Result<Uuid, String> {
        // ตรวจสอบ inventory ก่อน
        for item in &cmd.items {
            let available = self.inventory_service
                .check_availability(item.product_id, item.quantity)
                .await?;
                
            if !available {
                return Err(format!("Product {} is not available", item.product_id));
            }
        }
        
        // สร้าง Order aggregate
        let mut order = Order::new(cmd.customer_id);
        
        // เพิ่มสินค้า
        for item in &cmd.items {
            use rust_decimal::Decimal;
            use std::str::FromStr;
            
            let amount = Decimal::from_str(&item.unit_price_amount)
                .map_err(|_| "Invalid price".to_string())?;
            let price = Money::new(amount, Currency::THB)
                .map_err(|e| e.to_string())?;
            let qty = Quantity::new(item.quantity)
                .map_err(|e| e.to_string())?;
            
            order.add_item(item.product_id, item.product_name.clone(), qty, price)
                .map_err(|e| e.to_string())?;
        }
        
        // สร้าง shipping address
        let postal_code = PostalCode::new(&cmd.shipping_postal_code)
            .map_err(|e| e.to_string())?;
        let address = Address::new(
            cmd.shipping_street,
            cmd.shipping_city,
            postal_code,
            cmd.shipping_country,
        ).map_err(|e| e.to_string())?;
        
        // Place order
        order.place(address).map_err(|e| e.to_string())?;
        
        let order_id = order.id();
        
        // เก็บ order
        self.order_repository.save(&order)
            .await
            .map_err(|e| e.to_string())?;
        
        // Reserve inventory
        for item in &cmd.items {
            self.inventory_service
                .reserve(item.product_id, item.quantity)
                .await?;
        }
        
        // Publish domain events
        let events = order.take_events();
        // หมายเหตุ: ต้องการ mut order แต่ take_events ต้องการ &mut self
        // ในทางปฏิบัติควร handle ก่อน save
        self.event_publisher.publish(events).await?;
        
        Ok(order_id)
    }
}
```

## Event Sourcing Basics

```rust
// src/domain/event_store.rs
use async_trait::async_trait;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredEvent {
    pub id: Uuid,
    pub aggregate_id: Uuid,
    pub aggregate_type: String,
    pub event_type: String,
    pub event_data: serde_json::Value,
    pub version: u64,
    pub occurred_at: DateTime<Utc>,
}

#[async_trait]
pub trait EventStore: Send + Sync {
    async fn append(
        &self,
        aggregate_id: Uuid,
        events: Vec<StoredEvent>,
        expected_version: Option<u64>,
    ) -> Result<(), EventStoreError>;
    
    async fn load(
        &self,
        aggregate_id: Uuid,
        from_version: Option<u64>,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;
    
    async fn load_all(
        &self,
        aggregate_type: &str,
        from_position: Option<u64>,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;
}

#[derive(Debug, thiserror::Error)]
pub enum EventStoreError {
    #[error("Optimistic concurrency conflict")]
    ConcurrencyConflict,
    #[error("Aggregate not found: {0}")]
    AggregateNotFound(Uuid),
    #[error("Storage error: {0}")]
    StorageError(String),
}
```

## CQRS Overview

```rust
// src/application/cqrs/mod.rs

// Command side
pub mod commands {
    use async_trait::async_trait;
    
    pub trait Command: Send + Sync {}
    
    #[async_trait]
    pub trait CommandHandler<C: Command> {
        type Output;
        type Error;
        
        async fn handle(&self, command: C) -> Result<Self::Output, Self::Error>;
    }
}

// Query side  
pub mod queries {
    use async_trait::async_trait;
    
    pub trait Query: Send + Sync {}
    
    #[async_trait]
    pub trait QueryHandler<Q: Query> {
        type Output;
        type Error;
        
        async fn handle(&self, query: Q) -> Result<Self::Output, Self::Error>;
    }
}

// Example: Place Order Command
use commands::{Command, CommandHandler};

pub struct PlaceOrderCommand {
    pub customer_id: uuid::Uuid,
    pub items: Vec<PlaceOrderItem>,
}

pub struct PlaceOrderItem {
    pub product_id: uuid::Uuid,
    pub quantity: u32,
}

impl Command for PlaceOrderCommand {}

// Example: Get Order Query
use queries::{Query, QueryHandler};

pub struct GetOrderQuery {
    pub order_id: uuid::Uuid,
}

impl Query for GetOrderQuery {}

#[derive(serde::Serialize)]
pub struct OrderReadModel {
    pub id: uuid::Uuid,
    pub customer_name: String,
    pub status: String,
    pub total_amount: String,
    pub item_count: u32,
}
```

## HTTP Handler

```rust
// src/interface/http/order_handler.rs
use actix_web::{web, HttpResponse};
use std::sync::Arc;
use uuid::Uuid;

use crate::application::order_service::{OrderApplicationService, PlaceOrderCommand, OrderItemCommand};

pub async fn place_order(
    service: web::Data<Arc<OrderApplicationService>>,
    body: web::Json<PlaceOrderRequest>,
) -> HttpResponse {
    let cmd = PlaceOrderCommand {
        customer_id: body.customer_id,
        items: body.items.iter().map(|i| OrderItemCommand {
            product_id: i.product_id,
            product_name: i.product_name.clone(),
            quantity: i.quantity,
            unit_price_amount: i.unit_price.clone(),
        }).collect(),
        shipping_street: body.shipping_address.street.clone(),
        shipping_city: body.shipping_address.city.clone(),
        shipping_postal_code: body.shipping_address.postal_code.clone(),
        shipping_country: body.shipping_address.country.clone(),
    };
    
    match service.place_order(cmd).await {
        Ok(order_id) => HttpResponse::Created().json(serde_json::json!({
            "order_id": order_id
        })),
        Err(e) => HttpResponse::BadRequest().json(serde_json::json!({
            "error": e
        })),
    }
}

#[derive(serde::Deserialize)]
pub struct PlaceOrderRequest {
    pub customer_id: Uuid,
    pub items: Vec<OrderItemRequest>,
    pub shipping_address: ShippingAddressRequest,
}

#[derive(serde::Deserialize)]
pub struct OrderItemRequest {
    pub product_id: Uuid,
    pub product_name: String,
    pub quantity: u32,
    pub unit_price: String,
}

#[derive(serde::Deserialize)]
pub struct ShippingAddressRequest {
    pub street: String,
    pub city: String,
    pub postal_code: String,
    pub country: String,
}
```

## Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use rust_decimal::Decimal;
    use uuid::Uuid;
    
    fn create_test_order() -> Order {
        Order::new(Uuid::new_v4())
    }
    
    fn create_test_item() -> (Uuid, String, Quantity, Money) {
        let product_id = Uuid::new_v4();
        let quantity = Quantity::new(2).unwrap();
        let price = Money::new(Decimal::from(100), Currency::THB).unwrap();
        (product_id, "Test Product".to_string(), quantity, price)
    }
    
    #[test]
    fn test_add_item_to_draft_order() {
        let mut order = create_test_order();
        let (pid, name, qty, price) = create_test_item();
        
        assert!(order.add_item(pid, name, qty, price).is_ok());
        assert_eq!(order.items().len(), 1);
    }
    
    #[test]
    fn test_cannot_add_item_to_placed_order() {
        let mut order = create_test_order();
        let (pid, name, qty, price) = create_test_item();
        
        order.add_item(pid, name.clone(), qty.clone(), price.clone()).unwrap();
        
        let postal = PostalCode::new("10110").unwrap();
        let addr = Address::new(
            "123 Test St".to_string(),
            "Bangkok".to_string(),
            postal,
            "Thailand".to_string(),
        ).unwrap();
        
        order.place(addr).unwrap();
        
        let result = order.add_item(Uuid::new_v4(), name, qty, price);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_cannot_place_empty_order() {
        let mut order = create_test_order();
        let postal = PostalCode::new("10110").unwrap();
        let addr = Address::new(
            "123 Test St".to_string(),
            "Bangkok".to_string(),
            postal,
            "Thailand".to_string(),
        ).unwrap();
        
        assert!(order.place(addr).is_err());
    }
    
    #[test]
    fn test_order_total() {
        let mut order = create_test_order();
        
        let qty1 = Quantity::new(2).unwrap();
        let price1 = Money::new(Decimal::from(100), Currency::THB).unwrap();
        order.add_item(Uuid::new_v4(), "Product 1".to_string(), qty1, price1).unwrap();
        
        let qty2 = Quantity::new(3).unwrap();
        let price2 = Money::new(Decimal::from(50), Currency::THB).unwrap();
        order.add_item(Uuid::new_v4(), "Product 2".to_string(), qty2, price2).unwrap();
        
        let total = order.total();
        assert_eq!(total.amount(), Decimal::from(350)); // 2*100 + 3*50 = 350
    }
    
    #[test]
    fn test_domain_events_generated() {
        let mut order = create_test_order();
        let (pid, name, qty, price) = create_test_item();
        
        order.add_item(pid, name, qty, price).unwrap();
        
        let postal = PostalCode::new("10110").unwrap();
        let addr = Address::new(
            "123 Test St".to_string(),
            "Bangkok".to_string(),
            postal,
            "Thailand".to_string(),
        ).unwrap();
        
        order.place(addr).unwrap();
        
        let events = order.take_events();
        // ควรมี ItemAdded และ OrderPlaced events
        assert!(events.len() >= 2);
    }
    
    #[test]
    fn test_money_addition() {
        let m1 = Money::new(Decimal::from(100), Currency::THB).unwrap();
        let m2 = Money::new(Decimal::from(200), Currency::THB).unwrap();
        
        let sum = m1.add(&m2).unwrap();
        assert_eq!(sum.amount(), Decimal::from(300));
    }
    
    #[test]
    fn test_money_currency_mismatch() {
        let m1 = Money::new(Decimal::from(100), Currency::THB).unwrap();
        let m2 = Money::new(Decimal::from(100), Currency::USD).unwrap();
        
        assert!(m1.add(&m2).is_err());
    }
    
    #[test]
    fn test_discount_pricing() {
        use crate::domain::services::PricingService;
        
        let price = Money::new(Decimal::from(1000), Currency::THB).unwrap();
        let discounted = PricingService::apply_discount(&price, Decimal::from(10)).unwrap();
        
        assert_eq!(discounted.amount(), Decimal::from(900));
    }
}
```

## สรุป

DDD ใน Rust มีจุดแข็งหลัก:

1. **Type System** - Value objects และ enums ช่วยทำให้ domain model ชัดเจน
2. **Ownership** - บังคับให้ domain logic อยู่ใน aggregate
3. **Traits** - ทำหน้าที่เป็น interface สำหรับ bounded context integration
4. **Pattern Matching** - จัดการ domain events ได้อย่างปลอดภัย

---

## Navigation

- [← Part 061: Clean Architecture](../part_061/README.md)
- [→ Part 063: CQRS Pattern](../part_063/README.md)
- [กลับหน้าหลัก](../../README.md)
