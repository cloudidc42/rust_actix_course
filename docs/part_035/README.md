# Part 035: Database Transactions

## สารบัญ
- [BEGIN/COMMIT/ROLLBACK](#begincommitrollback)
- [SQLx Transaction API](#sqlx-transaction-api)
- [Savepoints](#savepoints)
- [Transaction Isolation Levels](#transaction-isolation-levels)
- [Deadlock Detection](#deadlock-detection)
- [Nested Transactions](#nested-transactions)
- [Transaction ใน Repository Pattern](#transaction-ใน-repository-pattern)
- [Practical: Transfer Funds Example](#practical-transfer-funds-example)

---

## BEGIN/COMMIT/ROLLBACK

### แนวคิด Transaction

Transaction คือกลุ่มของ SQL operations ที่ต้องทำงานเป็นหน่วยเดียว (All or Nothing):
- **BEGIN**: เริ่มต้น transaction
- **COMMIT**: ยืนยัน การเปลี่ยนแปลงทั้งหมด
- **ROLLBACK**: ยกเลิก การเปลี่ยนแปลงทั้งหมด

คุณสมบัติ **ACID**:
- **A**tomicity: ทำทั้งหมดหรือไม่ทำเลย
- **C**onsistency: ข้อมูลสอดคล้องกันเสมอ
- **I**solation: transactions แยกออกจากกัน
- **D**urability: ถ้า commit แล้ว ข้อมูลถาวร

### SQL Transaction พื้นฐาน

```sql
-- ตัวอย่าง SQL transaction สำหรับโอนเงิน
BEGIN;

-- ตรวจสอบ balance
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- หักเงิน
UPDATE accounts 
SET balance = balance - 1000
WHERE id = 1 AND balance >= 1000;

-- เพิ่มเงิน
UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;

-- บันทึก transaction
INSERT INTO transactions (from_account, to_account, amount)
VALUES (1, 2, 1000);

COMMIT;  -- หรือ ROLLBACK ถ้ามีปัญหา
```

---

## SQLx Transaction API

### Transaction พื้นฐาน

```rust
// src/db/transactions.rs
use sqlx::{PgPool, Transaction, Postgres};
use uuid::Uuid;
use crate::errors::AppError;

// Transaction แบบ manual
pub async fn basic_transaction(pool: &PgPool) -> Result<(), AppError> {
    // เริ่ม transaction
    let mut tx: Transaction<'_, Postgres> = pool.begin()
        .await
        .map_err(AppError::Database)?;
    
    // ทำงานใน transaction
    let result = async {
        sqlx::query!(
            "INSERT INTO users (email, username, password_hash, display_name) 
             VALUES ($1, $2, $3, $4)",
            "user@example.com",
            "testuser",
            "hashed_password",
            "Test User"
        )
        .execute(&mut *tx)
        .await?;
        
        sqlx::query!(
            "INSERT INTO audit_logs (action, resource_type) 
             VALUES ($1, $2)",
            "user.created",
            "user"
        )
        .execute(&mut *tx)
        .await?;
        
        Ok::<(), sqlx::Error>(())
    }.await;
    
    // Commit หรือ Rollback ตามผลลัพธ์
    match result {
        Ok(_) => {
            tx.commit().await.map_err(AppError::Database)?;
            log::info!("Transaction committed successfully");
        }
        Err(e) => {
            tx.rollback().await.map_err(AppError::Database)?;
            log::error!("Transaction rolled back: {}", e);
            return Err(AppError::Database(e));
        }
    }
    
    Ok(())
}

// Transaction แบบ auto-rollback เมื่อ error
pub async fn auto_rollback_transaction(pool: &PgPool) -> Result<Uuid, AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // ถ้า function นี้ return Err → tx จะ drop → auto rollback
    let user_id = create_user_in_tx(&mut tx).await?;
    create_audit_in_tx(&mut tx, user_id).await?;
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(user_id)
}

async fn create_user_in_tx(
    tx: &mut Transaction<'_, Postgres>
) -> Result<Uuid, AppError> {
    let user = sqlx::query!(
        r#"
        INSERT INTO users (email, username, password_hash, display_name)
        VALUES ($1, $2, $3, $4)
        RETURNING id
        "#,
        "newuser@example.com",
        "newuser",
        "hashed_pw",
        "New User"
    )
    .fetch_one(&mut **tx)
    .await
    .map_err(AppError::Database)?;
    
    Ok(user.id)
}

async fn create_audit_in_tx(
    tx: &mut Transaction<'_, Postgres>,
    user_id: Uuid
) -> Result<(), AppError> {
    sqlx::query!(
        "INSERT INTO audit_logs (action, resource_type, resource_id) VALUES ($1, $2, $3)",
        "user.created",
        "user",
        user_id
    )
    .execute(&mut **tx)
    .await
    .map_err(AppError::Database)?;
    
    Ok(())
}
```

### Transaction ที่ส่งผ่าน Repository

```rust
// src/repositories/base.rs
use sqlx::{PgPool, Transaction, Postgres, Executor};

// Trait สำหรับ executor (ทั้ง pool และ transaction)
pub trait DbExecutor<'e>: Executor<'e, Database = Postgres> {}
impl<'e> DbExecutor<'e> for &'e PgPool {}
impl<'e> DbExecutor<'e> for &'e mut Transaction<'_, Postgres> {}

// Repository ที่รับ executor ได้ทั้ง pool และ transaction
pub struct UserRepository;

impl UserRepository {
    pub async fn create_user<'e, E: DbExecutor<'e>>(
        executor: E,
        email: &str,
        username: &str,
        password_hash: &str,
    ) -> Result<Uuid, sqlx::Error> {
        let user = sqlx::query!(
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id
            "#,
            email,
            username,
            password_hash,
            username  // display_name = username by default
        )
        .fetch_one(executor)
        .await?;
        
        Ok(user.id)
    }
}

// ใช้งาน:
pub async fn example_usage(pool: &PgPool) -> Result<(), sqlx::Error> {
    // ใช้ pool โดยตรง
    let _id = UserRepository::create_user(
        pool, "user@example.com", "user1", "hash"
    ).await?;
    
    // ใช้ใน transaction
    let mut tx = pool.begin().await?;
    let _id = UserRepository::create_user(
        &mut *tx, "user2@example.com", "user2", "hash"
    ).await?;
    tx.commit().await?;
    
    Ok(())
}
```

---

## Savepoints

### การใช้ Savepoints

```rust
// src/db/savepoints.rs

// Savepoint ช่วยให้ rollback ไป point ที่กำหนดได้
// โดยไม่ต้อง rollback ทั้ง transaction
pub async fn transaction_with_savepoints(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // Step 1: Insert user (required)
    sqlx::query!(
        "INSERT INTO users (email, username, password_hash, display_name) 
         VALUES ('a@b.com', 'abc', 'hash', 'ABC')"
    )
    .execute(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // สร้าง savepoint ก่อน optional operations
    sqlx::query!("SAVEPOINT sp1")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    // Step 2: Optional operation (อาจ fail ได้)
    let optional_result = sqlx::query!(
        "INSERT INTO categories (name, slug) VALUES ('TestCat', 'test-cat')"
    )
    .execute(&mut *tx)
    .await;
    
    match optional_result {
        Ok(_) => {
            // ทำงานสำเร็จ → release savepoint
            sqlx::query!("RELEASE SAVEPOINT sp1")
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
            log::info!("Optional step succeeded");
        }
        Err(e) => {
            // ล้มเหลว → rollback ไป savepoint (ไม่ rollback ทั้งหมด)
            sqlx::query!("ROLLBACK TO SAVEPOINT sp1")
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
            log::warn!("Optional step failed, continuing: {}", e);
        }
    }
    
    // Step 3: Continue with main operations
    sqlx::query!(
        "INSERT INTO audit_logs (action, resource_type) VALUES ('test', 'test')"
    )
    .execute(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // Commit ทั้งหมด (รวมถึง step 1 และ step 3 ถ้า step 2 ล้มเหลว)
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}
```

---

## Transaction Isolation Levels

### Isolation Levels ใน PostgreSQL

```rust
// src/db/isolation.rs

// PostgreSQL มี 4 isolation levels:
// READ UNCOMMITTED (same as READ COMMITTED in PostgreSQL)
// READ COMMITTED (default)
// REPEATABLE READ
// SERIALIZABLE

pub async fn transaction_with_isolation(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // ตั้ง isolation level
    sqlx::query!("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    // หรือ READ COMMITTED (default)
    // sqlx::query!("SET TRANSACTION ISOLATION LEVEL READ COMMITTED")
    
    // หรือ REPEATABLE READ
    // sqlx::query!("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
    
    // ทำงาน
    sqlx::query!("SELECT 1")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}

// READ COMMITTED: สำหรับ normal operations
// - ป้องกัน dirty reads
// - อาจเห็น non-repeatable reads
pub async fn read_committed_example(pool: &PgPool) -> Result<i64, AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // อ่านข้อมูล
    let count1 = sqlx::query_scalar!(
        r#"SELECT COUNT(*) as "count!" FROM users WHERE is_active = true"#
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // ถ้า user อื่น insert ระหว่างนี้ → count2 อาจต่างจาก count1
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    
    let count2 = sqlx::query_scalar!(
        r#"SELECT COUNT(*) as "count!" FROM users WHERE is_active = true"#
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(count2)
}

// REPEATABLE READ: สำหรับ operations ที่ต้องการ consistent reads
// - ป้องกัน dirty reads และ non-repeatable reads
// - อาจเห็น phantom reads (แต่ PostgreSQL ป้องกันด้วย)
pub async fn repeatable_read_example(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    sqlx::query!("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    // อ่านข้อมูลครั้งแรก
    let balance1 = sqlx::query_scalar!(
        r#"SELECT balance as "balance!" FROM accounts WHERE id = 1"#
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // แม้ user อื่น update ระหว่างนี้ → เราจะเห็น balance เดิม
    tokio::time::sleep(std::time::Duration::from_millis(100)).await;
    
    let balance2 = sqlx::query_scalar!(
        r#"SELECT balance as "balance!" FROM accounts WHERE id = 1"#
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    assert_eq!(balance1, balance2, "Should be same under REPEATABLE READ");
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}

// SERIALIZABLE: สำหรับ critical operations
// - Highest isolation
// - อาจมี serialization errors → ต้อง retry
pub async fn serializable_with_retry(
    pool: &PgPool,
    max_retries: u32
) -> Result<(), AppError> {
    let mut attempts = 0;
    
    loop {
        attempts += 1;
        
        let result = serializable_transaction(pool).await;
        
        match result {
            Ok(_) => return Ok(()),
            Err(AppError::Database(e)) => {
                // Serialization failure → retry
                if let Some(db_error) = e.as_database_error() {
                    if db_error.code().map_or(false, |c| c == "40001") {
                        if attempts < max_retries {
                            log::warn!(
                                "Serialization failure, retrying {}/{}",
                                attempts,
                                max_retries
                            );
                            tokio::time::sleep(
                                std::time::Duration::from_millis(50 * attempts as u64)
                            ).await;
                            continue;
                        }
                    }
                }
                return Err(AppError::Database(e));
            }
            Err(e) => return Err(e),
        }
    }
}

async fn serializable_transaction(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    sqlx::query!("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    // Critical operations...
    sqlx::query!("SELECT 1").execute(&mut *tx).await.map_err(AppError::Database)?;
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}
```

---

## Deadlock Detection

### การจัดการ Deadlock

```rust
// src/db/deadlock.rs

// Deadlock เกิดเมื่อ 2 transactions รอกันเป็น circle:
// T1 lock A, รอ B
// T2 lock B, รอ A
// → Deadlock!

// PostgreSQL detect deadlock อัตโนมัติและ abort transaction หนึ่ง

pub async fn handle_deadlock(pool: &PgPool) -> Result<(), AppError> {
    let max_retries = 5;
    let mut attempts = 0;
    
    loop {
        attempts += 1;
        
        match do_transaction(pool).await {
            Ok(_) => {
                log::info!("Transaction succeeded after {} attempts", attempts);
                return Ok(());
            }
            Err(AppError::Database(e)) => {
                if is_deadlock_error(&e) && attempts < max_retries {
                    let delay = std::time::Duration::from_millis(
                        100 * (1 << attempts) as u64  // Exponential backoff
                    );
                    log::warn!(
                        "Deadlock detected (attempt {}/{}), retrying in {:?}",
                        attempts,
                        max_retries,
                        delay
                    );
                    tokio::time::sleep(delay).await;
                    continue;
                }
                return Err(AppError::Database(e));
            }
            Err(e) => return Err(e),
        }
    }
}

fn is_deadlock_error(error: &sqlx::Error) -> bool {
    if let Some(db_error) = error.as_database_error() {
        // PostgreSQL deadlock error code: 40P01
        // Serialization failure: 40001
        let code = db_error.code().map(|c| c.to_string());
        matches!(code.as_deref(), Some("40P01") | Some("40001"))
    } else {
        false
    }
}

async fn do_transaction(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // ล็อค rows ด้วย SELECT FOR UPDATE
    // ใช้ consistent ordering เพื่อหลีกเลี่ยง deadlock
    let _row = sqlx::query!(
        "SELECT id FROM accounts WHERE id = $1 FOR UPDATE",
        Uuid::new_v4()  // always lock in same order (smallest id first)
    )
    .fetch_optional(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // ทำงาน...
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}

// Strategy: Lock rows ใน consistent order เพื่อป้องกัน deadlock
pub async fn lock_accounts_in_order(
    pool: &PgPool,
    account1: Uuid,
    account2: Uuid,
    amount: i64,
) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // Sort IDs → lock ใน consistent order เสมอ
    let (first, second) = if account1 < account2 {
        (account1, account2)
    } else {
        (account2, account1)
    };
    
    // Lock first account
    sqlx::query!(
        "SELECT id, balance FROM accounts WHERE id = $1 FOR UPDATE",
        first
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // Lock second account
    sqlx::query!(
        "SELECT id, balance FROM accounts WHERE id = $1 FOR UPDATE",
        second
    )
    .fetch_one(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // โอนเงิน
    sqlx::query!(
        "UPDATE accounts SET balance = balance - $2 WHERE id = $1",
        account1, amount
    )
    .execute(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    sqlx::query!(
        "UPDATE accounts SET balance = balance + $2 WHERE id = $1",
        account2, amount
    )
    .execute(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}
```

---

## Nested Transactions

### Nested Transaction ด้วย Savepoints

```rust
// src/db/nested_tx.rs

// SQLx ไม่รองรับ "true" nested transactions
// แต่ใช้ Savepoints แทนได้

pub struct NestedTransaction<'a> {
    tx: &'a mut sqlx::Transaction<'static, sqlx::Postgres>,
    savepoint_name: String,
    committed: bool,
}

impl<'a> NestedTransaction<'a> {
    pub async fn begin(
        tx: &'a mut sqlx::Transaction<'static, sqlx::Postgres>,
        name: &str
    ) -> Result<Self, sqlx::Error> {
        sqlx::query(&format!("SAVEPOINT {}", name))
            .execute(&mut **tx)
            .await?;
        
        Ok(Self {
            tx,
            savepoint_name: name.to_string(),
            committed: false,
        })
    }
    
    pub async fn commit(mut self) -> Result<(), sqlx::Error> {
        sqlx::query(&format!("RELEASE SAVEPOINT {}", self.savepoint_name))
            .execute(&mut **self.tx)
            .await?;
        self.committed = true;
        Ok(())
    }
    
    pub async fn rollback(self) -> Result<(), sqlx::Error> {
        sqlx::query(&format!("ROLLBACK TO SAVEPOINT {}", self.savepoint_name))
            .execute(&mut **self.tx)
            .await?;
        Ok(())
    }
}

// ตัวอย่างการใช้งาน nested transaction concept
pub async fn nested_operations_example(pool: &PgPool) -> Result<(), AppError> {
    let mut tx = pool.begin().await.map_err(AppError::Database)?;
    
    // Main transaction operations
    sqlx::query!(
        "INSERT INTO users (email, username, password_hash, display_name) 
         VALUES ('main@test.com', 'mainuser', 'hash', 'Main User')"
    )
    .execute(&mut *tx)
    .await
    .map_err(AppError::Database)?;
    
    // "Nested" transaction ด้วย savepoint
    sqlx::query!("SAVEPOINT inner_tx")
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
    
    let inner_result = sqlx::query!(
        "INSERT INTO categories (name, slug) VALUES ('TestCat2', 'test-cat-2')"
    )
    .execute(&mut *tx)
    .await;
    
    match inner_result {
        Ok(_) => {
            sqlx::query!("RELEASE SAVEPOINT inner_tx")
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
        }
        Err(_) => {
            sqlx::query!("ROLLBACK TO SAVEPOINT inner_tx")
                .execute(&mut *tx)
                .await
                .map_err(AppError::Database)?;
            // Continue main transaction
        }
    }
    
    // Continue main transaction...
    tx.commit().await.map_err(AppError::Database)?;
    
    Ok(())
}
```

---

## Transaction ใน Repository Pattern

### Repository ที่รองรับ Transaction

```rust
// src/repositories/transactional.rs
use sqlx::{PgPool, Transaction, Postgres};
use uuid::Uuid;

// Trait สำหรับ repository ที่รองรับ transaction
#[async_trait::async_trait]
pub trait UserRepo {
    async fn create(
        &self,
        email: &str,
        username: &str,
        password_hash: &str,
    ) -> Result<Uuid, AppError>;
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserRecord>, AppError>;
}

pub struct UserRecord {
    pub id: Uuid,
    pub email: String,
    pub username: String,
}

// Implementation ที่ใช้ Pool
pub struct PgUserRepo {
    pool: PgPool,
}

#[async_trait::async_trait]
impl UserRepo for PgUserRepo {
    async fn create(
        &self,
        email: &str,
        username: &str,
        password_hash: &str,
    ) -> Result<Uuid, AppError> {
        let user = sqlx::query!(
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id
            "#,
            email, username, password_hash, username
        )
        .fetch_one(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user.id)
    }
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<UserRecord>, AppError> {
        let user = sqlx::query_as!(
            UserRecord,
            "SELECT id, email, username FROM users WHERE id = $1",
            id
        )
        .fetch_optional(&self.pool)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user)
    }
}

// Unit of Work pattern
pub struct UnitOfWork<'a> {
    tx: Transaction<'a, Postgres>,
}

impl<'a> UnitOfWork<'a> {
    pub async fn new(pool: &'a PgPool) -> Result<Self, AppError> {
        let tx = pool.begin().await.map_err(AppError::Database)?;
        Ok(Self { tx })
    }
    
    pub async fn create_user(
        &mut self,
        email: &str,
        username: &str,
        password_hash: &str,
    ) -> Result<Uuid, AppError> {
        let user = sqlx::query!(
            r#"
            INSERT INTO users (email, username, password_hash, display_name)
            VALUES ($1, $2, $3, $4)
            RETURNING id
            "#,
            email, username, password_hash, username
        )
        .fetch_one(&mut *self.tx)
        .await
        .map_err(AppError::Database)?;
        
        Ok(user.id)
    }
    
    pub async fn create_audit_log(
        &mut self,
        action: &str,
        resource_type: &str,
        resource_id: Option<Uuid>,
    ) -> Result<(), AppError> {
        sqlx::query!(
            "INSERT INTO audit_logs (action, resource_type, resource_id) VALUES ($1, $2, $3)",
            action,
            resource_type,
            resource_id
        )
        .execute(&mut *self.tx)
        .await
        .map_err(AppError::Database)?;
        
        Ok(())
    }
    
    pub async fn commit(self) -> Result<(), AppError> {
        self.tx.commit().await.map_err(AppError::Database)
    }
    
    pub async fn rollback(self) -> Result<(), AppError> {
        self.tx.rollback().await.map_err(AppError::Database)
    }
}

// Service ที่ใช้ Unit of Work
pub struct UserService {
    pool: PgPool,
}

impl UserService {
    pub async fn register_user(
        &self,
        email: &str,
        username: &str,
        password_hash: &str,
    ) -> Result<Uuid, AppError> {
        let mut uow = UnitOfWork::new(&self.pool).await?;
        
        let user_id = match uow.create_user(email, username, password_hash).await {
            Ok(id) => id,
            Err(e) => {
                uow.rollback().await?;
                return Err(e);
            }
        };
        
        match uow.create_audit_log("user.registered", "user", Some(user_id)).await {
            Ok(_) => {}
            Err(e) => {
                uow.rollback().await?;
                return Err(e);
            }
        }
        
        uow.commit().await?;
        
        Ok(user_id)
    }
}
```

---

## Practical: Transfer Funds Example

### ระบบโอนเงินสมบูรณ์

```rust
// src/services/transfer_service.rs

use sqlx::PgPool;
use uuid::Uuid;
use serde::{Deserialize, Serialize};
use chrono::{DateTime, Utc};

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct Account {
    pub id: Uuid,
    pub user_id: Uuid,
    pub account_number: String,
    pub balance: i64,  // ใช้ cents เพื่อหลีกเลี่ยง floating point
    pub currency: String,
    pub is_active: bool,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
pub struct TransactionRecord {
    pub id: Uuid,
    pub from_account_id: Uuid,
    pub to_account_id: Uuid,
    pub amount: i64,
    pub currency: String,
    pub description: Option<String>,
    pub status: String,
    pub reference_id: String,
    pub created_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct TransferRequest {
    pub from_account_id: Uuid,
    pub to_account_id: Uuid,
    pub amount: i64,  // ใน cents
    pub description: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct TransferResult {
    pub transaction_id: Uuid,
    pub from_balance: i64,
    pub to_balance: i64,
    pub status: String,
}

pub struct TransferService {
    pool: PgPool,
}

impl TransferService {
    pub fn new(pool: PgPool) -> Self {
        Self { pool }
    }
    
    pub async fn transfer(
        &self,
        request: TransferRequest,
    ) -> Result<TransferResult, AppError> {
        // Validate
        if request.amount <= 0 {
            return Err(AppError::BadRequest("Amount must be positive".to_string()));
        }
        
        if request.from_account_id == request.to_account_id {
            return Err(AppError::BadRequest("Cannot transfer to same account".to_string()));
        }
        
        // Execute with retry สำหรับ deadlock/serialization errors
        let max_retries = 3;
        
        for attempt in 1..=max_retries {
            match self.execute_transfer(&request).await {
                Ok(result) => return Ok(result),
                Err(AppError::Database(ref e)) if is_retryable_error(e) && attempt < max_retries => {
                    let delay = std::time::Duration::from_millis(50 * attempt as u64);
                    log::warn!("Retryable error on attempt {}: {}", attempt, e);
                    tokio::time::sleep(delay).await;
                    continue;
                }
                Err(e) => return Err(e),
            }
        }
        
        unreachable!()
    }
    
    async fn execute_transfer(
        &self,
        request: &TransferRequest,
    ) -> Result<TransferResult, AppError> {
        let mut tx = self.pool.begin().await.map_err(AppError::Database)?;
        
        // Set serializable isolation
        sqlx::query!("SET TRANSACTION ISOLATION LEVEL REPEATABLE READ")
            .execute(&mut *tx)
            .await
            .map_err(AppError::Database)?;
        
        // Lock accounts ใน consistent order (ป้องกัน deadlock)
        let (first_id, second_id) = if request.from_account_id < request.to_account_id {
            (request.from_account_id, request.to_account_id)
        } else {
            (request.to_account_id, request.from_account_id)
        };
        
        // Lock first account
        let first_account = sqlx::query_as!(
            Account,
            r#"
            SELECT id, user_id, account_number, balance, currency, is_active, created_at, updated_at
            FROM accounts
            WHERE id = $1
            FOR UPDATE
            "#,
            first_id
        )
        .fetch_optional(&mut *tx)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound(format!("Account {} not found", first_id)))?;
        
        // Lock second account
        let second_account = sqlx::query_as!(
            Account,
            r#"
            SELECT id, user_id, account_number, balance, currency, is_active, created_at, updated_at
            FROM accounts
            WHERE id = $1
            FOR UPDATE
            "#,
            second_id
        )
        .fetch_optional(&mut *tx)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::NotFound(format!("Account {} not found", second_id)))?;
        
        // Map back to from/to
        let (from_account, to_account) = if first_id == request.from_account_id {
            (first_account, second_account)
        } else {
            (second_account, first_account)
        };
        
        // Validate accounts
        if !from_account.is_active {
            tx.rollback().await.ok();
            return Err(AppError::BadRequest("Source account is not active".to_string()));
        }
        
        if !to_account.is_active {
            tx.rollback().await.ok();
            return Err(AppError::BadRequest("Destination account is not active".to_string()));
        }
        
        if from_account.currency != to_account.currency {
            tx.rollback().await.ok();
            return Err(AppError::BadRequest("Currency mismatch".to_string()));
        }
        
        if from_account.balance < request.amount {
            tx.rollback().await.ok();
            return Err(AppError::BadRequest(format!(
                "Insufficient balance. Available: {}, Required: {}",
                from_account.balance,
                request.amount
            )));
        }
        
        // Deduct from source
        let from_new = sqlx::query!(
            r#"
            UPDATE accounts
            SET balance = balance - $2, updated_at = NOW()
            WHERE id = $1 AND balance >= $2
            RETURNING balance
            "#,
            from_account.id,
            request.amount
        )
        .fetch_optional(&mut *tx)
        .await
        .map_err(AppError::Database)?
        .ok_or(AppError::BadRequest("Insufficient balance (concurrent update)".to_string()))?;
        
        // Add to destination
        let to_new = sqlx::query!(
            r#"
            UPDATE accounts
            SET balance = balance + $2, updated_at = NOW()
            WHERE id = $1
            RETURNING balance
            "#,
            to_account.id,
            request.amount
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // Create transaction record
        let reference_id = format!("TXN-{}", uuid::Uuid::new_v4().to_string().replace('-', "")[..12].to_uppercase());
        
        let tx_record = sqlx::query!(
            r#"
            INSERT INTO transactions (
                from_account_id, to_account_id, amount, currency,
                description, status, reference_id
            )
            VALUES ($1, $2, $3, $4, $5, 'completed', $6)
            RETURNING id
            "#,
            from_account.id,
            to_account.id,
            request.amount,
            from_account.currency,
            request.description,
            reference_id
        )
        .fetch_one(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // Log the transfer
        sqlx::query!(
            r#"
            INSERT INTO audit_logs (action, resource_type, resource_id, new_data)
            VALUES ('transfer.completed', 'transaction', $1, $2)
            "#,
            tx_record.id,
            serde_json::json!({
                "from": from_account.id,
                "to": to_account.id,
                "amount": request.amount,
                "reference": reference_id
            })
        )
        .execute(&mut *tx)
        .await
        .map_err(AppError::Database)?;
        
        // Commit
        tx.commit().await.map_err(AppError::Database)?;
        
        Ok(TransferResult {
            transaction_id: tx_record.id,
            from_balance: from_new.balance,
            to_balance: to_new.balance,
            status: "completed".to_string(),
        })
    }
}

fn is_retryable_error(error: &sqlx::Error) -> bool {
    if let Some(db_error) = error.as_database_error() {
        let code = db_error.code().map(|c| c.to_string());
        matches!(code.as_deref(), Some("40001") | Some("40P01"))
    } else {
        false
    }
}
```

### Handler สำหรับ Transfer API

```rust
// src/handlers/transfer.rs
use actix_web::{post, web, HttpResponse};
use sqlx::PgPool;

#[post("/api/v1/transfers")]
pub async fn create_transfer(
    pool: web::Data<PgPool>,
    request: web::Json<TransferRequest>,
) -> HttpResponse {
    let service = TransferService::new(pool.get_ref().clone());
    
    match service.transfer(request.into_inner()).await {
        Ok(result) => HttpResponse::Ok().json(result),
        Err(AppError::BadRequest(msg)) => {
            HttpResponse::BadRequest().json(serde_json::json!({
                "error": msg
            }))
        }
        Err(AppError::NotFound(msg)) => {
            HttpResponse::NotFound().json(serde_json::json!({
                "error": msg
            }))
        }
        Err(e) => {
            log::error!("Transfer error: {}", e);
            HttpResponse::InternalServerError().json(serde_json::json!({
                "error": "Transfer failed"
            }))
        }
    }
}
```

---

## Navigation

| ก่อนหน้า | หน้าหลัก | ถัดไป |
|---------|---------|------|
| [Part 034: Connection Pooling](../part_034/README.md) | [README หลัก](../../README.md) | [Part 036: Redis Integration](../part_036/README.md) |
