# Part 064: Event Sourcing in Rust

## ภาพรวม

Event Sourcing เป็น pattern ที่เก็บ state ของ application โดยใช้ sequence ของ events แทนการเก็บ current state ตรง ๆ บทนี้จะสร้างระบบ Account Balance Tracking ด้วย Event Sourcing

## แนวคิดหลัก

```
Traditional:           Event Sourcing:
┌─────────┐           ┌─────────────────────────────┐
│ Account │           │ AccountCreated(id, owner)   │
│         │    vs     │ MoneyDeposited(100)          │
│ bal=350 │           │ MoneyWithdrawn(50)           │
└─────────┘           │ MoneyDeposited(300)          │
                      │ → Reconstruct: bal = 350     │
                      └─────────────────────────────┘
```

## Cargo.toml

```toml
[package]
name = "event-sourcing"
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
sqlx = { version = "0.7", features = ["postgres", "runtime-tokio", "uuid", "chrono", "json"] }
```

## Event Types

```rust
// src/events/account_events.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

/// ทุก event ต้องมี metadata เหล่านี้
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct EventMetadata {
    pub event_id: Uuid,
    pub aggregate_id: Uuid,
    pub aggregate_type: String,
    pub event_type: String,
    pub version: u64,
    pub occurred_at: DateTime<Utc>,
    pub correlation_id: Option<Uuid>,
    pub causation_id: Option<Uuid>,
}

/// Account events
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
pub enum AccountEvent {
    AccountOpened(AccountOpenedData),
    MoneyDeposited(MoneyDepositedData),
    MoneyWithdrawn(MoneyWithdrawnData),
    AccountFrozen(AccountFrozenData),
    AccountClosed(AccountClosedData),
    TransferInitiated(TransferInitiatedData),
    TransferCompleted(TransferCompletedData),
    TransferFailed(TransferFailedData),
}

impl AccountEvent {
    pub fn event_type(&self) -> &'static str {
        match self {
            AccountEvent::AccountOpened(_) => "AccountOpened",
            AccountEvent::MoneyDeposited(_) => "MoneyDeposited",
            AccountEvent::MoneyWithdrawn(_) => "MoneyWithdrawn",
            AccountEvent::AccountFrozen(_) => "AccountFrozen",
            AccountEvent::AccountClosed(_) => "AccountClosed",
            AccountEvent::TransferInitiated(_) => "TransferInitiated",
            AccountEvent::TransferCompleted(_) => "TransferCompleted",
            AccountEvent::TransferFailed(_) => "TransferFailed",
        }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccountOpenedData {
    pub owner_id: Uuid,
    pub owner_name: String,
    pub initial_balance: f64,
    pub account_type: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MoneyDepositedData {
    pub amount: f64,
    pub reference: String,
    pub description: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MoneyWithdrawnData {
    pub amount: f64,
    pub reference: String,
    pub description: Option<String>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccountFrozenData {
    pub reason: String,
    pub frozen_by: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccountClosedData {
    pub reason: String,
    pub closed_by: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransferInitiatedData {
    pub transfer_id: Uuid,
    pub to_account_id: Uuid,
    pub amount: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransferCompletedData {
    pub transfer_id: Uuid,
    pub amount: f64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct TransferFailedData {
    pub transfer_id: Uuid,
    pub reason: String,
}

/// Stored event ใน event store
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct StoredEvent {
    pub metadata: EventMetadata,
    pub payload: serde_json::Value,
}
```

## Aggregate - Account

```rust
// src/aggregates/account.rs
use uuid::Uuid;
use thiserror::Error;
use crate::events::account_events::*;

#[derive(Debug, Clone, PartialEq)]
pub enum AccountStatus {
    Active,
    Frozen,
    Closed,
}

#[derive(Debug, Error)]
pub enum AccountError {
    #[error("Insufficient balance: have {available}, need {required}")]
    InsufficientBalance { available: f64, required: f64 },
    #[error("Account is {0:?}")]
    InvalidStatus(AccountStatus),
    #[error("Deposit amount must be positive: {0}")]
    InvalidDepositAmount(f64),
    #[error("Withdrawal amount must be positive: {0}")]
    InvalidWithdrawalAmount(f64),
    #[error("Account already closed")]
    AlreadyClosed,
    #[error("Transfer not found: {0}")]
    TransferNotFound(Uuid),
}

/// Account aggregate - ถูก reconstruct จาก events
pub struct Account {
    pub id: Uuid,
    pub owner_id: Uuid,
    pub owner_name: String,
    pub balance: f64,
    pub status: AccountStatus,
    pub version: u64,
    // Pending events ที่ยังไม่ได้บันทึก
    pending_events: Vec<AccountEvent>,
}

impl Account {
    /// สร้าง Account ใหม่ (generate event)
    pub fn open(
        owner_id: Uuid,
        owner_name: String,
        initial_balance: f64,
        account_type: String,
    ) -> Result<Self, AccountError> {
        if initial_balance < 0.0 {
            return Err(AccountError::InvalidDepositAmount(initial_balance));
        }
        
        let id = Uuid::new_v4();
        let event = AccountEvent::AccountOpened(AccountOpenedData {
            owner_id,
            owner_name: owner_name.clone(),
            initial_balance,
            account_type,
        });
        
        let mut account = Account {
            id,
            owner_id,
            owner_name,
            balance: 0.0,
            status: AccountStatus::Active,
            version: 0,
            pending_events: Vec::new(),
        };
        
        // Apply event เพื่ออัพเดท state
        account.apply_event(&event);
        account.pending_events.push(event);
        
        Ok(account)
    }
    
    /// Reconstruct Account จาก events (Event Sourcing หลัก)
    pub fn from_events(id: Uuid, events: Vec<StoredEvent>) -> Result<Self, String> {
        if events.is_empty() {
            return Err(format!("No events for account {}", id));
        }
        
        let mut account = Account {
            id,
            owner_id: Uuid::nil(),
            owner_name: String::new(),
            balance: 0.0,
            status: AccountStatus::Active,
            version: 0,
            pending_events: Vec::new(),
        };
        
        for stored_event in events {
            let event: AccountEvent = serde_json::from_value(stored_event.payload)
                .map_err(|e| format!("Failed to deserialize event: {}", e))?;
            account.apply_event(&event);
            account.version = stored_event.metadata.version;
        }
        
        Ok(account)
    }
    
    /// Apply event เพื่อเปลี่ยน state (ไม่มี side effects)
    fn apply_event(&mut self, event: &AccountEvent) {
        match event {
            AccountEvent::AccountOpened(data) => {
                self.owner_id = data.owner_id;
                self.owner_name = data.owner_name.clone();
                self.balance = data.initial_balance;
                self.status = AccountStatus::Active;
            }
            AccountEvent::MoneyDeposited(data) => {
                self.balance += data.amount;
            }
            AccountEvent::MoneyWithdrawn(data) => {
                self.balance -= data.amount;
            }
            AccountEvent::AccountFrozen(_) => {
                self.status = AccountStatus::Frozen;
            }
            AccountEvent::AccountClosed(_) => {
                self.status = AccountStatus::Closed;
            }
            AccountEvent::TransferInitiated(data) => {
                // Reserve amount
                self.balance -= data.amount;
            }
            AccountEvent::TransferCompleted(_) => {
                // Already deducted in TransferInitiated
            }
            AccountEvent::TransferFailed(data) => {
                // Refund the amount
                // ต้องหา amount จาก TransferInitiated event
                // ในทางปฏิบัติ ควรเก็บ amount ใน TransferFailedData
            }
        }
    }
    
    /// Deposit money
    pub fn deposit(
        &mut self,
        amount: f64,
        reference: String,
        description: Option<String>,
    ) -> Result<(), AccountError> {
        if self.status != AccountStatus::Active {
            return Err(AccountError::InvalidStatus(self.status.clone()));
        }
        if amount <= 0.0 {
            return Err(AccountError::InvalidDepositAmount(amount));
        }
        
        let event = AccountEvent::MoneyDeposited(MoneyDepositedData {
            amount,
            reference,
            description,
        });
        
        self.apply_event(&event);
        self.pending_events.push(event);
        Ok(())
    }
    
    /// Withdraw money
    pub fn withdraw(
        &mut self,
        amount: f64,
        reference: String,
        description: Option<String>,
    ) -> Result<(), AccountError> {
        if self.status != AccountStatus::Active {
            return Err(AccountError::InvalidStatus(self.status.clone()));
        }
        if amount <= 0.0 {
            return Err(AccountError::InvalidWithdrawalAmount(amount));
        }
        if self.balance < amount {
            return Err(AccountError::InsufficientBalance {
                available: self.balance,
                required: amount,
            });
        }
        
        let event = AccountEvent::MoneyWithdrawn(MoneyWithdrawnData {
            amount,
            reference,
            description,
        });
        
        self.apply_event(&event);
        self.pending_events.push(event);
        Ok(())
    }
    
    /// Freeze account
    pub fn freeze(&mut self, reason: String, frozen_by: String) -> Result<(), AccountError> {
        if self.status == AccountStatus::Closed {
            return Err(AccountError::AlreadyClosed);
        }
        
        let event = AccountEvent::AccountFrozen(AccountFrozenData { reason, frozen_by });
        self.apply_event(&event);
        self.pending_events.push(event);
        Ok(())
    }
    
    /// Close account
    pub fn close(&mut self, reason: String, closed_by: String) -> Result<(), AccountError> {
        if self.status == AccountStatus::Closed {
            return Err(AccountError::AlreadyClosed);
        }
        
        let event = AccountEvent::AccountClosed(AccountClosedData { reason, closed_by });
        self.apply_event(&event);
        self.pending_events.push(event);
        Ok(())
    }
    
    pub fn take_pending_events(&mut self) -> Vec<AccountEvent> {
        std::mem::take(&mut self.pending_events)
    }
}
```

## Event Store Implementation

```rust
// src/event_store/postgres_event_store.rs
use async_trait::async_trait;
use sqlx::PgPool;
use uuid::Uuid;
use chrono::Utc;
use serde_json::json;

use crate::events::account_events::{StoredEvent, EventMetadata, AccountEvent};

#[async_trait]
pub trait EventStore: Send + Sync {
    async fn append_events(
        &self,
        aggregate_id: Uuid,
        aggregate_type: &str,
        events: Vec<AccountEvent>,
        expected_version: u64,
    ) -> Result<u64, EventStoreError>;
    
    async fn load_events(
        &self,
        aggregate_id: Uuid,
        from_version: Option<u64>,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;
    
    async fn load_all_events(
        &self,
        aggregate_type: &str,
        from_position: Option<u64>,
        limit: u32,
    ) -> Result<Vec<StoredEvent>, EventStoreError>;
}

#[derive(Debug, thiserror::Error)]
pub enum EventStoreError {
    #[error("Optimistic concurrency conflict: expected version {expected}, got {actual}")]
    ConcurrencyConflict { expected: u64, actual: u64 },
    #[error("Database error: {0}")]
    DatabaseError(String),
    #[error("Serialization error: {0}")]
    SerializationError(String),
}

pub struct PostgresEventStore {
    pool: PgPool,
}

impl PostgresEventStore {
    pub fn new(pool: PgPool) -> Self {
        PostgresEventStore { pool }
    }
}

#[async_trait]
impl EventStore for PostgresEventStore {
    async fn append_events(
        &self,
        aggregate_id: Uuid,
        aggregate_type: &str,
        events: Vec<AccountEvent>,
        expected_version: u64,
    ) -> Result<u64, EventStoreError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?;
        
        // ตรวจสอบ version (optimistic locking)
        let current_version: i64 = sqlx::query_scalar!(
            "SELECT COALESCE(MAX(version), 0) FROM events WHERE aggregate_id = $1",
            aggregate_id
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?
        .unwrap_or(0);
        
        if current_version as u64 != expected_version {
            return Err(EventStoreError::ConcurrencyConflict {
                expected: expected_version,
                actual: current_version as u64,
            });
        }
        
        let mut version = expected_version;
        
        for event in events {
            version += 1;
            let event_type = event.event_type();
            let payload = serde_json::to_value(&event)
                .map_err(|e| EventStoreError::SerializationError(e.to_string()))?;
            
            sqlx::query!(
                r#"
                INSERT INTO events (
                    event_id, aggregate_id, aggregate_type, event_type,
                    version, payload, occurred_at
                )
                VALUES ($1, $2, $3, $4, $5, $6, $7)
                "#,
                Uuid::new_v4(),
                aggregate_id,
                aggregate_type,
                event_type,
                version as i64,
                payload,
                Utc::now(),
            )
            .execute(&mut *tx)
            .await
            .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?;
        }
        
        tx.commit().await
            .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?;
        
        Ok(version)
    }
    
    async fn load_events(
        &self,
        aggregate_id: Uuid,
        from_version: Option<u64>,
    ) -> Result<Vec<StoredEvent>, EventStoreError> {
        let from_v = from_version.unwrap_or(0) as i64;
        
        let rows = sqlx::query!(
            r#"
            SELECT event_id, aggregate_id, aggregate_type, event_type,
                   version, payload, occurred_at
            FROM events
            WHERE aggregate_id = $1 AND version > $2
            ORDER BY version ASC
            "#,
            aggregate_id,
            from_v
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?;
        
        let events = rows.into_iter().map(|row| {
            StoredEvent {
                metadata: EventMetadata {
                    event_id: row.event_id,
                    aggregate_id: row.aggregate_id,
                    aggregate_type: row.aggregate_type,
                    event_type: row.event_type,
                    version: row.version as u64,
                    occurred_at: row.occurred_at,
                    correlation_id: None,
                    causation_id: None,
                },
                payload: row.payload,
            }
        }).collect();
        
        Ok(events)
    }
    
    async fn load_all_events(
        &self,
        aggregate_type: &str,
        from_position: Option<u64>,
        limit: u32,
    ) -> Result<Vec<StoredEvent>, EventStoreError> {
        let from_pos = from_position.unwrap_or(0) as i64;
        
        let rows = sqlx::query!(
            r#"
            SELECT event_id, aggregate_id, aggregate_type, event_type,
                   version, payload, occurred_at
            FROM events
            WHERE aggregate_type = $1 AND global_sequence > $2
            ORDER BY global_sequence ASC
            LIMIT $3
            "#,
            aggregate_type,
            from_pos,
            limit as i64
        )
        .fetch_all(&self.pool)
        .await
        .map_err(|e| EventStoreError::DatabaseError(e.to_string()))?;
        
        let events = rows.into_iter().map(|row| {
            StoredEvent {
                metadata: EventMetadata {
                    event_id: row.event_id,
                    aggregate_id: row.aggregate_id,
                    aggregate_type: row.aggregate_type,
                    event_type: row.event_type,
                    version: row.version as u64,
                    occurred_at: row.occurred_at,
                    correlation_id: None,
                    causation_id: None,
                },
                payload: row.payload,
            }
        }).collect();
        
        Ok(events)
    }
}
```

## Snapshots สำหรับ Performance

```rust
// src/snapshots/account_snapshot.rs
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

/// Snapshot เก็บ state ณ เวลาหนึ่ง ไม่ต้อง replay ทุก events
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AccountSnapshot {
    pub aggregate_id: Uuid,
    pub version: u64,
    pub balance: f64,
    pub owner_id: Uuid,
    pub owner_name: String,
    pub status: String,
    pub created_at: DateTime<Utc>,
}

pub struct SnapshotStore {
    pool: PgPool,
}

impl SnapshotStore {
    pub fn new(pool: PgPool) -> Self {
        SnapshotStore { pool }
    }
    
    pub async fn save_snapshot(&self, snapshot: &AccountSnapshot) -> Result<(), sqlx::Error> {
        let data = serde_json::to_value(snapshot).unwrap();
        
        sqlx::query!(
            r#"
            INSERT INTO snapshots (aggregate_id, version, data, created_at)
            VALUES ($1, $2, $3, $4)
            ON CONFLICT (aggregate_id) DO UPDATE
            SET version = EXCLUDED.version,
                data = EXCLUDED.data,
                created_at = EXCLUDED.created_at
            "#,
            snapshot.aggregate_id,
            snapshot.version as i64,
            data,
            snapshot.created_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
    
    pub async fn load_snapshot(&self, aggregate_id: Uuid) -> Result<Option<AccountSnapshot>, sqlx::Error> {
        let row = sqlx::query!(
            "SELECT data FROM snapshots WHERE aggregate_id = $1",
            aggregate_id
        )
        .fetch_optional(&self.pool)
        .await?;
        
        match row {
            Some(r) => {
                let snapshot: AccountSnapshot = serde_json::from_value(r.data).unwrap();
                Ok(Some(snapshot))
            }
            None => Ok(None),
        }
    }
}

const SNAPSHOT_THRESHOLD: u64 = 100; // snapshot ทุก 100 events

pub struct AccountRepository {
    event_store: std::sync::Arc<dyn EventStore>,
    snapshot_store: SnapshotStore,
}

impl AccountRepository {
    pub async fn load(&self, account_id: Uuid) -> Result<Option<Account>, String> {
        // ลอง load snapshot ก่อน
        let (start_version, mut account_opt) = match self.snapshot_store
            .load_snapshot(account_id)
            .await
            .map_err(|e| e.to_string())?
        {
            Some(snapshot) => {
                let account = Account::from_snapshot(&snapshot)?;
                (snapshot.version, Some(account))
            }
            None => (0, None),
        };
        
        // Load events หลังจาก snapshot
        let events = self.event_store
            .load_events(account_id, Some(start_version))
            .await
            .map_err(|e| e.to_string())?;
        
        if events.is_empty() && account_opt.is_none() {
            return Ok(None);
        }
        
        // Apply events หลัง snapshot
        let account = match account_opt {
            Some(mut acc) => {
                for event in events {
                    let ae: AccountEvent = serde_json::from_value(event.payload)
                        .map_err(|e| e.to_string())?;
                    acc.apply_event(&ae);
                    acc.version = event.metadata.version;
                }
                acc
            }
            None => Account::from_events(account_id, events)?,
        };
        
        Ok(Some(account))
    }
    
    pub async fn save(&self, account: &mut Account) -> Result<(), String> {
        let events = account.take_pending_events();
        if events.is_empty() {
            return Ok(());
        }
        
        let new_version = self.event_store
            .append_events(account.id, "Account", events, account.version)
            .await
            .map_err(|e| e.to_string())?;
        
        account.version = new_version;
        
        // สร้าง snapshot ถ้า events เยอะพอ
        if new_version % SNAPSHOT_THRESHOLD == 0 {
            let snapshot = AccountSnapshot {
                aggregate_id: account.id,
                version: new_version,
                balance: account.balance,
                owner_id: account.owner_id,
                owner_name: account.owner_name.clone(),
                status: format!("{:?}", account.status),
                created_at: Utc::now(),
            };
            
            self.snapshot_store.save_snapshot(&snapshot).await
                .map_err(|e| e.to_string())?;
        }
        
        Ok(())
    }
}
```

## Projections และ Read Models

```rust
// src/projections/account_balance_projection.rs
use sqlx::PgPool;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use serde::{Deserialize, Serialize};

use crate::events::account_events::*;

/// Read model สำหรับ account balance history
#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct AccountBalanceReadModel {
    pub account_id: Uuid,
    pub owner_name: String,
    pub current_balance: f64,
    pub total_deposits: f64,
    pub total_withdrawals: f64,
    pub transaction_count: i64,
    pub last_transaction_at: Option<DateTime<Utc>>,
    pub status: String,
}

/// Transaction history read model
#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct TransactionReadModel {
    pub id: Uuid,
    pub account_id: Uuid,
    pub transaction_type: String,
    pub amount: f64,
    pub balance_after: f64,
    pub reference: String,
    pub description: Option<String>,
    pub occurred_at: DateTime<Utc>,
}

pub struct AccountBalanceProjection {
    pool: PgPool,
}

impl AccountBalanceProjection {
    pub fn new(pool: PgPool) -> Self {
        AccountBalanceProjection { pool }
    }
    
    pub async fn project(&self, event: &StoredEvent) -> Result<(), sqlx::Error> {
        let account_id = event.metadata.aggregate_id;
        let account_event: AccountEvent = serde_json::from_value(event.payload.clone())?;
        
        match account_event {
            AccountEvent::AccountOpened(data) => {
                sqlx::query!(
                    r#"
                    INSERT INTO account_balances (
                        account_id, owner_name, current_balance,
                        total_deposits, total_withdrawals, transaction_count, status
                    )
                    VALUES ($1, $2, $3, $3, 0, 1, 'ACTIVE')
                    "#,
                    account_id,
                    data.owner_name,
                    data.initial_balance,
                )
                .execute(&self.pool)
                .await?;
            }
            
            AccountEvent::MoneyDeposited(data) => {
                sqlx::query!(
                    r#"
                    UPDATE account_balances
                    SET current_balance = current_balance + $2,
                        total_deposits = total_deposits + $2,
                        transaction_count = transaction_count + 1,
                        last_transaction_at = NOW()
                    WHERE account_id = $1
                    "#,
                    account_id,
                    data.amount,
                )
                .execute(&self.pool)
                .await?;
                
                // บันทึก transaction
                self.insert_transaction(
                    account_id,
                    "DEPOSIT",
                    data.amount,
                    data.reference,
                    data.description,
                    event.metadata.occurred_at,
                ).await?;
            }
            
            AccountEvent::MoneyWithdrawn(data) => {
                sqlx::query!(
                    r#"
                    UPDATE account_balances
                    SET current_balance = current_balance - $2,
                        total_withdrawals = total_withdrawals + $2,
                        transaction_count = transaction_count + 1,
                        last_transaction_at = NOW()
                    WHERE account_id = $1
                    "#,
                    account_id,
                    data.amount,
                )
                .execute(&self.pool)
                .await?;
                
                self.insert_transaction(
                    account_id,
                    "WITHDRAWAL",
                    data.amount,
                    data.reference,
                    data.description,
                    event.metadata.occurred_at,
                ).await?;
            }
            
            AccountEvent::AccountFrozen(_) => {
                sqlx::query!(
                    "UPDATE account_balances SET status = 'FROZEN' WHERE account_id = $1",
                    account_id
                )
                .execute(&self.pool)
                .await?;
            }
            
            AccountEvent::AccountClosed(_) => {
                sqlx::query!(
                    "UPDATE account_balances SET status = 'CLOSED' WHERE account_id = $1",
                    account_id
                )
                .execute(&self.pool)
                .await?;
            }
            
            _ => {}
        }
        
        Ok(())
    }
    
    async fn insert_transaction(
        &self,
        account_id: Uuid,
        tx_type: &str,
        amount: f64,
        reference: String,
        description: Option<String>,
        occurred_at: DateTime<Utc>,
    ) -> Result<(), sqlx::Error> {
        // ดึง balance หลัง transaction
        let balance: f64 = sqlx::query_scalar!(
            "SELECT current_balance FROM account_balances WHERE account_id = $1",
            account_id
        )
        .fetch_one(&self.pool)
        .await?;
        
        sqlx::query!(
            r#"
            INSERT INTO transactions (
                id, account_id, transaction_type, amount, balance_after,
                reference, description, occurred_at
            )
            VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
            "#,
            Uuid::new_v4(),
            account_id,
            tx_type,
            amount,
            balance,
            reference,
            description,
            occurred_at,
        )
        .execute(&self.pool)
        .await?;
        
        Ok(())
    }
}
```

## Event Replay

```rust
// src/replay/event_replayer.rs
use std::sync::Arc;
use uuid::Uuid;

use crate::{
    event_store::EventStore,
    projections::AccountBalanceProjection,
};

/// Event Replayer - สร้าง read model ใหม่จาก events ทั้งหมด
pub struct EventReplayer {
    event_store: Arc<dyn EventStore>,
    projection: Arc<AccountBalanceProjection>,
}

impl EventReplayer {
    pub fn new(
        event_store: Arc<dyn EventStore>,
        projection: Arc<AccountBalanceProjection>,
    ) -> Self {
        EventReplayer { event_store, projection }
    }
    
    /// Replay ทุก events ของ aggregate type นี้
    pub async fn replay_all(&self, aggregate_type: &str) -> Result<u64, String> {
        let mut position = 0u64;
        let mut processed = 0u64;
        let batch_size = 100u32;
        
        tracing::info!("Starting event replay for {}", aggregate_type);
        
        loop {
            let events = self.event_store
                .load_all_events(aggregate_type, Some(position), batch_size)
                .await
                .map_err(|e| e.to_string())?;
            
            if events.is_empty() {
                break;
            }
            
            for event in &events {
                self.projection.project(event).await
                    .map_err(|e| e.to_string())?;
                
                position = event.metadata.version;
                processed += 1;
            }
            
            tracing::info!("Processed {} events", processed);
            
            if events.len() < batch_size as usize {
                break;
            }
        }
        
        tracing::info!("Event replay completed. Total processed: {}", processed);
        Ok(processed)
    }
    
    /// Replay events สำหรับ aggregate เดียว
    pub async fn replay_aggregate(&self, aggregate_id: Uuid) -> Result<(), String> {
        let events = self.event_store
            .load_events(aggregate_id, None)
            .await
            .map_err(|e| e.to_string())?;
        
        for event in &events {
            self.projection.project(event).await
                .map_err(|e| e.to_string())?;
        }
        
        Ok(())
    }
}
```

## HTTP Handlers

```rust
// src/interface/http/account_handler.rs
use actix_web::{web, HttpResponse};
use std::sync::Arc;
use uuid::Uuid;
use serde::{Deserialize, Serialize};

use crate::{
    aggregates::Account,
    snapshots::AccountRepository,
    projections::{AccountBalanceReadModel, TransactionReadModel},
};

pub struct AccountHandlerState {
    pub repository: Arc<AccountRepository>,
    pub read_repo: Arc<dyn AccountReadRepository>,
}

#[derive(Deserialize)]
pub struct OpenAccountRequest {
    pub owner_name: String,
    pub initial_balance: f64,
    pub account_type: String,
}

#[derive(Deserialize)]
pub struct DepositRequest {
    pub amount: f64,
    pub reference: String,
    pub description: Option<String>,
}

#[derive(Deserialize)]
pub struct WithdrawRequest {
    pub amount: f64,
    pub reference: String,
    pub description: Option<String>,
}

pub async fn open_account(
    state: web::Data<AccountHandlerState>,
    body: web::Json<OpenAccountRequest>,
) -> HttpResponse {
    let owner_id = Uuid::new_v4(); // normally from auth token
    
    let mut account = match Account::open(
        owner_id,
        body.owner_name.clone(),
        body.initial_balance,
        body.account_type.clone(),
    ) {
        Ok(a) => a,
        Err(e) => return HttpResponse::BadRequest().json(serde_json::json!({"error": e.to_string()})),
    };
    
    match state.repository.save(&mut account).await {
        Ok(_) => HttpResponse::Created().json(serde_json::json!({
            "account_id": account.id,
            "balance": account.balance,
        })),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

pub async fn deposit(
    state: web::Data<AccountHandlerState>,
    path: web::Path<Uuid>,
    body: web::Json<DepositRequest>,
) -> HttpResponse {
    let account_id = *path;
    
    let mut account = match state.repository.load(account_id).await {
        Ok(Some(a)) => a,
        Ok(None) => return HttpResponse::NotFound().finish(),
        Err(e) => return HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    };
    
    if let Err(e) = account.deposit(body.amount, body.reference.clone(), body.description.clone()) {
        return HttpResponse::BadRequest().json(serde_json::json!({"error": e.to_string()}));
    }
    
    match state.repository.save(&mut account).await {
        Ok(_) => HttpResponse::Ok().json(serde_json::json!({
            "balance": account.balance,
        })),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

pub async fn get_balance(
    state: web::Data<AccountHandlerState>,
    path: web::Path<Uuid>,
) -> HttpResponse {
    match state.read_repo.find_balance(*path).await {
        Ok(Some(balance)) => HttpResponse::Ok().json(balance),
        Ok(None) => HttpResponse::NotFound().finish(),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

pub async fn get_transaction_history(
    state: web::Data<AccountHandlerState>,
    path: web::Path<Uuid>,
    params: web::Query<PaginationParams>,
) -> HttpResponse {
    match state.read_repo
        .find_transactions(*path, params.page.unwrap_or(1), params.per_page.unwrap_or(20))
        .await
    {
        Ok(transactions) => HttpResponse::Ok().json(transactions),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({"error": e})),
    }
}

#[derive(Deserialize)]
pub struct PaginationParams {
    pub page: Option<u32>,
    pub per_page: Option<u32>,
}

#[async_trait::async_trait]
pub trait AccountReadRepository: Send + Sync {
    async fn find_balance(&self, account_id: Uuid) -> Result<Option<AccountBalanceReadModel>, String>;
    async fn find_transactions(
        &self,
        account_id: Uuid,
        page: u32,
        per_page: u32,
    ) -> Result<Vec<TransactionReadModel>, String>;
}
```

## Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use uuid::Uuid;
    
    #[test]
    fn test_open_account() {
        let owner_id = Uuid::new_v4();
        let account = Account::open(
            owner_id,
            "John Doe".to_string(),
            1000.0,
            "SAVINGS".to_string(),
        ).unwrap();
        
        assert_eq!(account.balance, 1000.0);
        assert_eq!(account.owner_name, "John Doe");
        assert_eq!(account.status, AccountStatus::Active);
        assert_eq!(account.version, 0);
    }
    
    #[test]
    fn test_deposit() {
        let owner_id = Uuid::new_v4();
        let mut account = Account::open(
            owner_id,
            "Jane".to_string(),
            500.0,
            "CHECKING".to_string(),
        ).unwrap();
        
        account.deposit(200.0, "REF001".to_string(), None).unwrap();
        assert_eq!(account.balance, 700.0);
    }
    
    #[test]
    fn test_withdraw() {
        let owner_id = Uuid::new_v4();
        let mut account = Account::open(
            owner_id,
            "Bob".to_string(),
            500.0,
            "CHECKING".to_string(),
        ).unwrap();
        
        account.withdraw(200.0, "REF002".to_string(), None).unwrap();
        assert_eq!(account.balance, 300.0);
    }
    
    #[test]
    fn test_insufficient_balance() {
        let owner_id = Uuid::new_v4();
        let mut account = Account::open(
            owner_id,
            "Alice".to_string(),
            100.0,
            "CHECKING".to_string(),
        ).unwrap();
        
        let result = account.withdraw(200.0, "REF003".to_string(), None);
        assert!(matches!(result, Err(AccountError::InsufficientBalance { .. })));
    }
    
    #[test]
    fn test_freeze_account() {
        let owner_id = Uuid::new_v4();
        let mut account = Account::open(
            owner_id,
            "Charlie".to_string(),
            500.0,
            "SAVINGS".to_string(),
        ).unwrap();
        
        account.freeze("Suspicious activity".to_string(), "admin".to_string()).unwrap();
        assert_eq!(account.status, AccountStatus::Frozen);
        
        let result = account.deposit(100.0, "REF004".to_string(), None);
        assert!(matches!(result, Err(AccountError::InvalidStatus(_))));
    }
    
    #[test]
    fn test_pending_events() {
        let owner_id = Uuid::new_v4();
        let mut account = Account::open(
            owner_id,
            "Test User".to_string(),
            500.0,
            "SAVINGS".to_string(),
        ).unwrap();
        
        account.deposit(100.0, "R1".to_string(), None).unwrap();
        account.withdraw(50.0, "R2".to_string(), None).unwrap();
        
        let events = account.take_pending_events();
        // AccountOpened + MoneyDeposited + MoneyWithdrawn = 3 events
        assert_eq!(events.len(), 3);
        
        // หลัง take แล้วควรว่าง
        let events2 = account.take_pending_events();
        assert!(events2.is_empty());
    }
}
```

## Database Schema

```sql
-- migrations/001_create_events_table.sql

CREATE TABLE events (
    global_sequence BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    version BIGINT NOT NULL,
    payload JSONB NOT NULL,
    correlation_id UUID,
    causation_id UUID,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON events(aggregate_id, version);
CREATE INDEX idx_events_type ON events(aggregate_type, global_sequence);

-- Snapshots table
CREATE TABLE snapshots (
    aggregate_id UUID PRIMARY KEY,
    version BIGINT NOT NULL,
    data JSONB NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Read model tables
CREATE TABLE account_balances (
    account_id UUID PRIMARY KEY,
    owner_name VARCHAR(200) NOT NULL,
    current_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    total_deposits DECIMAL(15,2) NOT NULL DEFAULT 0,
    total_withdrawals DECIMAL(15,2) NOT NULL DEFAULT 0,
    transaction_count BIGINT NOT NULL DEFAULT 0,
    last_transaction_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);

CREATE TABLE transactions (
    id UUID PRIMARY KEY,
    account_id UUID NOT NULL REFERENCES account_balances(account_id),
    transaction_type VARCHAR(20) NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    reference VARCHAR(100) NOT NULL,
    description TEXT,
    occurred_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_transactions_account ON transactions(account_id, occurred_at DESC);
```

## สรุป

Event Sourcing มีประโยชน์หลัก:

1. **Audit Trail สมบูรณ์** - ทราบทุก state change
2. **Time Travel** - สามารถ reconstruct state ณ เวลาใด ๆ ก็ได้
3. **Event Replay** - สร้าง read model ใหม่ได้เสมอ
4. **Debugging** - trace ปัญหาได้ง่าย

ข้อระวัง: Event schema migration ซับซ้อน, storage เพิ่มขึ้น

---

## Navigation

- [← Part 063: CQRS Pattern](../part_063/README.md)
- [→ Part 065: Microservices Architecture](../part_065/README.md)
- [กลับหน้าหลัก](../../README.md)
