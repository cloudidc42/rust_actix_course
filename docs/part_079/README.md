# Part 079: Testing Strategies

## บทนำ

การ test เป็นส่วนสำคัญของการพัฒนา software ที่มีคุณภาพ บทนี้จะครอบคลุมกลยุทธ์การ test ตั้งแต่ unit tests ไปจนถึง end-to-end testing สำหรับ Rust + Actix-web

## 1. Unit Tests for Business Logic

### 1.1 Basic Unit Tests

```rust
// src/services/product_service.rs
use crate::models::{Product, CreateProduct};

#[derive(Debug, Clone)]
pub struct ProductService {
    // dependencies
}

impl ProductService {
    pub fn calculate_discount(price: f64, percentage: f64) -> f64 {
        if percentage < 0.0 || percentage > 100.0 {
            return price;
        }
        price * (1.0 - percentage / 100.0)
    }
    
    pub fn validate_product(product: &CreateProduct) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        
        if product.name.is_empty() {
            errors.push("Name cannot be empty".to_string());
        }
        
        if product.name.len() > 255 {
            errors.push("Name cannot exceed 255 characters".to_string());
        }
        
        if product.price <= 0.0 {
            errors.push("Price must be greater than 0".to_string());
        }
        
        if product.stock < 0 {
            errors.push("Stock cannot be negative".to_string());
        }
        
        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
    
    pub fn calculate_pagination(total: i64, page: i64, per_page: i64) -> (i32, bool, bool) {
        let total_pages = ((total as f64) / (per_page as f64)).ceil() as i32;
        let has_next = page < total_pages as i64;
        let has_prev = page > 1;
        (total_pages, has_next, has_prev)
    }
}

// Unit tests
#[cfg(test)]
mod tests {
    use super::*;
    use crate::models::CreateProduct;
    
    // Test calculate_discount
    #[test]
    fn test_calculate_discount_valid() {
        let result = ProductService::calculate_discount(100.0, 20.0);
        assert_eq!(result, 80.0);
    }
    
    #[test]
    fn test_calculate_discount_zero_percent() {
        let result = ProductService::calculate_discount(100.0, 0.0);
        assert_eq!(result, 100.0);
    }
    
    #[test]
    fn test_calculate_discount_full_discount() {
        let result = ProductService::calculate_discount(100.0, 100.0);
        assert_eq!(result, 0.0);
    }
    
    #[test]
    fn test_calculate_discount_invalid_percentage() {
        // ถ้า percentage > 100, คืนราคาเต็ม
        let result = ProductService::calculate_discount(100.0, 150.0);
        assert_eq!(result, 100.0);
        
        // ถ้า percentage < 0, คืนราคาเต็ม
        let result = ProductService::calculate_discount(100.0, -10.0);
        assert_eq!(result, 100.0);
    }
    
    // Test validate_product
    #[test]
    fn test_validate_product_valid() {
        let product = CreateProduct {
            name: "Test Product".to_string(),
            price: 99.99,
            stock: 10,
            category: "test".to_string(),
            description: None,
            tags: vec![],
        };
        
        assert!(ProductService::validate_product(&product).is_ok());
    }
    
    #[test]
    fn test_validate_product_empty_name() {
        let product = CreateProduct {
            name: "".to_string(),
            price: 99.99,
            stock: 10,
            category: "test".to_string(),
            description: None,
            tags: vec![],
        };
        
        let result = ProductService::validate_product(&product);
        assert!(result.is_err());
        
        let errors = result.unwrap_err();
        assert!(errors.iter().any(|e| e.contains("Name cannot be empty")));
    }
    
    #[test]
    fn test_validate_product_negative_price() {
        let product = CreateProduct {
            name: "Test".to_string(),
            price: -10.0,
            stock: 10,
            category: "test".to_string(),
            description: None,
            tags: vec![],
        };
        
        let result = ProductService::validate_product(&product);
        assert!(result.is_err());
    }
    
    #[test]
    fn test_validate_product_multiple_errors() {
        let product = CreateProduct {
            name: "".to_string(),
            price: -10.0,
            stock: -5,
            category: "test".to_string(),
            description: None,
            tags: vec![],
        };
        
        let result = ProductService::validate_product(&product);
        let errors = result.unwrap_err();
        assert_eq!(errors.len(), 3);  // name, price, stock errors
    }
    
    // Test calculate_pagination
    #[test]
    fn test_pagination_first_page() {
        let (total_pages, has_next, has_prev) = 
            ProductService::calculate_pagination(100, 1, 20);
        
        assert_eq!(total_pages, 5);
        assert!(has_next);
        assert!(!has_prev);
    }
    
    #[test]
    fn test_pagination_last_page() {
        let (total_pages, has_next, has_prev) = 
            ProductService::calculate_pagination(100, 5, 20);
        
        assert_eq!(total_pages, 5);
        assert!(!has_next);
        assert!(has_prev);
    }
    
    #[test]
    fn test_pagination_partial_last_page() {
        // 101 items ÷ 20 per page = 6 pages (last page มี 1 item)
        let (total_pages, _, _) = 
            ProductService::calculate_pagination(101, 1, 20);
        
        assert_eq!(total_pages, 6);
    }
}
```

### 1.2 Test with Custom Assert Macros

```rust
// src/test_utils.rs
#[cfg(test)]
pub mod macros {
    /// assert ว่า error message มีข้อความที่ต้องการ
    #[macro_export]
    macro_rules! assert_error_contains {
        ($result:expr, $msg:expr) => {
            let errors = $result.unwrap_err();
            assert!(
                errors.iter().any(|e| e.contains($msg)),
                "Expected error containing '{}', got: {:?}",
                $msg,
                errors
            );
        };
    }
    
    /// assert ว่า f64 ใกล้เคียงกัน (สำหรับ floating point comparison)
    #[macro_export]
    macro_rules! assert_float_eq {
        ($a:expr, $b:expr) => {
            assert!(
                ($a - $b).abs() < 1e-10,
                "Expected {} to approximately equal {}",
                $a, $b
            );
        };
        ($a:expr, $b:expr, $tolerance:expr) => {
            assert!(
                ($a - $b).abs() < $tolerance,
                "Expected {} to equal {} within tolerance {}",
                $a, $b, $tolerance
            );
        };
    }
}
```

## 2. Integration Tests for Handlers

### 2.1 Setup Integration Test

```rust
// tests/common/mod.rs
use actix_web::{test, web, App, middleware};
use sqlx::PgPool;

pub async fn create_test_app(pool: PgPool) -> impl actix_web::dev::Service<
    actix_web::dev::ServiceRequest,
    Response = actix_web::dev::ServiceResponse,
    Error = actix_web::Error
> {
    test::init_service(
        App::new()
            .app_data(web::Data::new(pool))
            .app_data(
                web::JsonConfig::default()
                    .error_handler(|err, req| {
                        let response = actix_web::HttpResponse::BadRequest()
                            .json(serde_json::json!({
                                "error": err.to_string()
                            }));
                        actix_web::error::InternalError::from_response(err, response).into()
                    })
            )
            .service(
                web::scope("/api/v1")
                    .configure(crate::handlers::products::configure)
                    .configure(crate::handlers::users::configure)
            )
    ).await
}

pub fn get_auth_header(token: &str) -> (String, String) {
    ("Authorization".to_string(), format!("Bearer {}", token))
}
```

### 2.2 Integration Tests

```rust
// tests/products_test.rs
use actix_web::{test, http::StatusCode};
use serde_json::json;

mod common;

#[actix_web::test]
async fn test_list_products_returns_200() {
    let pool = setup_test_db().await;
    let app = common::create_test_app(pool).await;
    
    let req = test::TestRequest::get()
        .uri("/api/v1/products")
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    
    assert_eq!(resp.status(), StatusCode::OK);
    
    let body: serde_json::Value = test::read_body_json(resp).await;
    assert!(body["data"].is_array());
    assert!(body["meta"].is_object());
}

#[actix_web::test]
async fn test_get_product_not_found() {
    let pool = setup_test_db().await;
    let app = common::create_test_app(pool).await;
    
    let req = test::TestRequest::get()
        .uri("/api/v1/products/99999")
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    
    assert_eq!(resp.status(), StatusCode::NOT_FOUND);
    
    let body: serde_json::Value = test::read_body_json(resp).await;
    assert!(body["error"].is_string());
}

#[actix_web::test]
async fn test_create_product_success() {
    let pool = setup_test_db().await;
    let app = common::create_test_app(pool).await;
    
    let token = create_test_token("admin");
    
    let req = test::TestRequest::post()
        .uri("/api/v1/products")
        .insert_header(common::get_auth_header(&token))
        .insert_header(("Content-Type", "application/json"))
        .set_json(json!({
            "name": "Test Product",
            "price": 99.99,
            "stock": 10,
            "category": "test"
        }))
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    
    assert_eq!(resp.status(), StatusCode::CREATED);
    
    let body: serde_json::Value = test::read_body_json(resp).await;
    assert!(body["id"].is_number());
    assert_eq!(body["name"], "Test Product");
}

#[actix_web::test]
async fn test_create_product_validation_error() {
    let pool = setup_test_db().await;
    let app = common::create_test_app(pool).await;
    
    let token = create_test_token("admin");
    
    let req = test::TestRequest::post()
        .uri("/api/v1/products")
        .insert_header(common::get_auth_header(&token))
        .insert_header(("Content-Type", "application/json"))
        .set_json(json!({
            "name": "",         // invalid: empty name
            "price": -10.0,     // invalid: negative price
            "stock": 10,
            "category": "test"
        }))
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    
    assert_eq!(resp.status(), StatusCode::UNPROCESSABLE_ENTITY);
}

#[actix_web::test]
async fn test_create_product_unauthorized() {
    let pool = setup_test_db().await;
    let app = common::create_test_app(pool).await;
    
    // ไม่มี token
    let req = test::TestRequest::post()
        .uri("/api/v1/products")
        .insert_header(("Content-Type", "application/json"))
        .set_json(json!({
            "name": "Test",
            "price": 10.0,
            "stock": 1,
            "category": "test"
        }))
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    
    assert_eq!(resp.status(), StatusCode::UNAUTHORIZED);
}

async fn setup_test_db() -> PgPool {
    let database_url = std::env::var("TEST_DATABASE_URL")
        .unwrap_or_else(|_| "postgres://postgres:test@localhost/testdb".to_string());
    
    let pool = sqlx::postgres::PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to test database");
    
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");
    
    pool
}

fn create_test_token(role: &str) -> String {
    // สร้าง JWT token สำหรับ test
    use jsonwebtoken::{encode, Header, EncodingKey};
    use serde::{Deserialize, Serialize};
    
    #[derive(Serialize, Deserialize)]
    struct Claims {
        sub: String,
        role: String,
        exp: usize,
    }
    
    let claims = Claims {
        sub: "test-user-id".to_string(),
        role: role.to_string(),
        exp: (chrono::Utc::now() + chrono::Duration::hours(1)).timestamp() as usize,
    };
    
    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(b"test-secret")
    ).unwrap()
}
```

## 3. Database Tests (testcontainers)

### 3.1 Setup testcontainers

```toml
# Cargo.toml
[dev-dependencies]
testcontainers = "0.15"
testcontainers-modules = { version = "0.3", features = ["postgres", "redis"] }
```

### 3.2 Tests ด้วย testcontainers

```rust
// tests/db_tests.rs
use testcontainers::{clients::Cli, images::postgres::Postgres};
use testcontainers_modules::postgres;
use sqlx::PgPool;

struct TestDatabase {
    pool: PgPool,
    // container ต้องอยู่ใน scope เพื่อไม่ให้ถูก drop
    _container: testcontainers::Container<'static, Postgres>,
}

async fn setup_test_container() -> TestDatabase {
    let docker = Cli::default();
    let container = docker.run(postgres::Postgres::default());
    
    let host = container.get_host();
    let port = container.get_host_port_ipv4(5432);
    
    let database_url = format!(
        "postgres://postgres:postgres@{}:{}/postgres",
        host, port
    );
    
    let pool = PgPool::connect(&database_url)
        .await
        .expect("Failed to connect");
    
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Migration failed");
    
    TestDatabase { pool, _container: container }
}

#[tokio::test]
async fn test_create_and_retrieve_product() {
    let db = setup_test_container().await;
    
    // Create product
    let result = sqlx::query!(
        "INSERT INTO products (name, price, stock, category) VALUES ($1, $2, $3, $4) RETURNING id",
        "Test Product",
        99.99_f64,
        10,
        "electronics"
    )
    .fetch_one(&db.pool)
    .await
    .unwrap();
    
    let product_id = result.id;
    
    // Retrieve product
    let product = sqlx::query!(
        "SELECT * FROM products WHERE id = $1",
        product_id
    )
    .fetch_one(&db.pool)
    .await
    .unwrap();
    
    assert_eq!(product.name, "Test Product");
    assert_eq!(product.price, 99.99);
    assert_eq!(product.stock, 10);
}

#[tokio::test]
async fn test_update_product() {
    let db = setup_test_container().await;
    
    // Create
    let result = sqlx::query!(
        "INSERT INTO products (name, price, stock, category) VALUES ($1, $2, $3, $4) RETURNING id",
        "Old Name",
        50.0_f64,
        5,
        "test"
    )
    .fetch_one(&db.pool)
    .await
    .unwrap();
    
    // Update
    sqlx::query!(
        "UPDATE products SET name = $1, price = $2 WHERE id = $3",
        "New Name",
        75.0_f64,
        result.id
    )
    .execute(&db.pool)
    .await
    .unwrap();
    
    // Verify
    let updated = sqlx::query!(
        "SELECT name, price FROM products WHERE id = $1",
        result.id
    )
    .fetch_one(&db.pool)
    .await
    .unwrap();
    
    assert_eq!(updated.name, "New Name");
    assert_eq!(updated.price, 75.0);
}
```

## 4. Mock External Services

### 4.1 Trait-based Mocking

```rust
// src/services/email_service.rs
use async_trait::async_trait;

#[async_trait]
pub trait EmailService: Send + Sync {
    async fn send_welcome_email(&self, to: &str, name: &str) -> Result<(), EmailError>;
    async fn send_password_reset(&self, to: &str, token: &str) -> Result<(), EmailError>;
    async fn send_order_confirmation(&self, to: &str, order_id: i32) -> Result<(), EmailError>;
}

// Real implementation
pub struct SmtpEmailService {
    host: String,
    username: String,
    password: String,
}

#[async_trait]
impl EmailService for SmtpEmailService {
    async fn send_welcome_email(&self, to: &str, name: &str) -> Result<(), EmailError> {
        // Real SMTP implementation
        todo!()
    }
    
    async fn send_password_reset(&self, to: &str, token: &str) -> Result<(), EmailError> {
        todo!()
    }
    
    async fn send_order_confirmation(&self, to: &str, order_id: i32) -> Result<(), EmailError> {
        todo!()
    }
}

// Mock implementation สำหรับ tests
#[cfg(test)]
pub mod mocks {
    use super::*;
    use std::sync::{Arc, Mutex};
    
    pub struct MockEmailService {
        pub sent_emails: Arc<Mutex<Vec<SentEmail>>>,
        pub should_fail: bool,
    }
    
    #[derive(Debug, Clone)]
    pub struct SentEmail {
        pub to: String,
        pub email_type: String,
    }
    
    impl MockEmailService {
        pub fn new() -> Self {
            MockEmailService {
                sent_emails: Arc::new(Mutex::new(Vec::new())),
                should_fail: false,
            }
        }
        
        pub fn with_failure() -> Self {
            MockEmailService {
                sent_emails: Arc::new(Mutex::new(Vec::new())),
                should_fail: true,
            }
        }
        
        pub fn emails_sent(&self) -> Vec<SentEmail> {
            self.sent_emails.lock().unwrap().clone()
        }
    }
    
    #[async_trait]
    impl EmailService for MockEmailService {
        async fn send_welcome_email(&self, to: &str, _name: &str) -> Result<(), EmailError> {
            if self.should_fail {
                return Err(EmailError::SendFailed("Mock failure".to_string()));
            }
            self.sent_emails.lock().unwrap().push(SentEmail {
                to: to.to_string(),
                email_type: "welcome".to_string(),
            });
            Ok(())
        }
        
        async fn send_password_reset(&self, to: &str, _token: &str) -> Result<(), EmailError> {
            if self.should_fail {
                return Err(EmailError::SendFailed("Mock failure".to_string()));
            }
            self.sent_emails.lock().unwrap().push(SentEmail {
                to: to.to_string(),
                email_type: "password_reset".to_string(),
            });
            Ok(())
        }
        
        async fn send_order_confirmation(&self, to: &str, _order_id: i32) -> Result<(), EmailError> {
            if self.should_fail {
                return Err(EmailError::SendFailed("Mock failure".to_string()));
            }
            self.sent_emails.lock().unwrap().push(SentEmail {
                to: to.to_string(),
                email_type: "order_confirmation".to_string(),
            });
            Ok(())
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum EmailError {
    #[error("Failed to send email: {0}")]
    SendFailed(String),
    #[error("Invalid email address")]
    InvalidAddress,
}
```

### 4.2 Mock HTTP Service ด้วย wiremock

```toml
# Cargo.toml dev-dependencies
wiremock = "0.5"
```

```rust
// tests/external_api_test.rs
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path, header, body_json};
use serde_json::json;

#[tokio::test]
async fn test_payment_gateway_integration() {
    // Start mock server
    let mock_server = MockServer::start().await;
    
    // Setup mock expectation
    Mock::given(method("POST"))
        .and(path("/charge"))
        .and(header("Authorization", "Bearer test-key"))
        .and(body_json(json!({
            "amount": 9999,
            "currency": "THB",
            "source": "tok_test"
        })))
        .respond_with(ResponseTemplate::new(200)
            .set_json_body(json!({
                "id": "ch_test123",
                "status": "succeeded",
                "amount": 9999
            }))
        )
        .expect(1)
        .mount(&mock_server)
        .await;
    
    // Create service ที่ใช้ mock server URL
    let payment_service = PaymentService::new(mock_server.uri());
    
    // Test
    let result = payment_service.charge(
        9999,
        "THB",
        "tok_test",
        "test-key"
    ).await;
    
    assert!(result.is_ok());
    let charge = result.unwrap();
    assert_eq!(charge.id, "ch_test123");
    assert_eq!(charge.status, "succeeded");
    
    // Verify mock was called
    mock_server.verify().await;
}

#[tokio::test]
async fn test_payment_gateway_failure() {
    let mock_server = MockServer::start().await;
    
    Mock::given(method("POST"))
        .and(path("/charge"))
        .respond_with(ResponseTemplate::new(402)
            .set_json_body(json!({
                "error": "insufficient_funds",
                "message": "ยอดเงินไม่เพียงพอ"
            }))
        )
        .mount(&mock_server)
        .await;
    
    let payment_service = PaymentService::new(mock_server.uri());
    
    let result = payment_service.charge(
        9999,
        "THB",
        "tok_declined",
        "test-key"
    ).await;
    
    assert!(result.is_err());
    assert!(matches!(result.unwrap_err(), PaymentError::InsufficientFunds));
}
```

## 5. Test Coverage (cargo-tarpaulin)

### 5.1 Setup และการใช้งาน

```bash
# ติดตั้ง cargo-tarpaulin
cargo install cargo-tarpaulin

# Run coverage
cargo tarpaulin --verbose

# Coverage พร้อม output formats
cargo tarpaulin \
    --out Html \
    --out Xml \
    --out Json \
    --output-dir coverage/ \
    --verbose \
    --all-features \
    --workspace \
    --timeout 120

# ข้าม certain files
cargo tarpaulin \
    --exclude-files "src/main.rs" \
    --exclude-files "src/bin/*"

# ดู report
open coverage/tarpaulin-report.html
```

```toml
# .tarpaulin.toml
[report]
out = ["Html", "Xml"]
output-dir = "coverage"

[coverage]
exclude-files = [
    "src/main.rs",
    "src/bin/*",
    "tests/*",
]
timeout = 120
features = ["all"]
```

### 5.2 Coverage Badge

```yaml
# .github/workflows/coverage.yml
- name: Run coverage
  run: |
    cargo tarpaulin \
      --out Xml \
      --output-dir coverage/

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    files: coverage/cobertura.xml
```

## 6. Contract Testing

### 6.1 Consumer-Driven Contract Tests

```rust
// tests/contract_tests.rs
// ทดสอบว่า API response ตรงกับ contract ที่ตกลงกับ client

use serde_json::Value;

struct ApiContract {
    endpoint: String,
    method: String,
    expected_response: Value,
}

impl ApiContract {
    fn verify_response(&self, actual: &Value) -> Result<(), Vec<String>> {
        let mut errors = Vec::new();
        
        // ตรวจสอบว่ามี required fields ครบ
        self.check_fields(actual, &self.expected_response, &mut errors, "");
        
        if errors.is_empty() {
            Ok(())
        } else {
            Err(errors)
        }
    }
    
    fn check_fields(&self, actual: &Value, expected: &Value, errors: &mut Vec<String>, path: &str) {
        match expected {
            Value::Object(expected_obj) => {
                for (key, expected_val) in expected_obj {
                    let field_path = if path.is_empty() {
                        key.clone()
                    } else {
                        format!("{}.{}", path, key)
                    };
                    
                    if let Some(actual_val) = actual.get(key) {
                        // Recursive check
                        self.check_fields(actual_val, expected_val, errors, &field_path);
                    } else {
                        errors.push(format!("Missing field: {}", field_path));
                    }
                }
            }
            Value::String(expected_type) if expected_type.starts_with("type:") => {
                let expected_type = expected_type.trim_start_matches("type:");
                let actual_type = match actual {
                    Value::String(_) => "string",
                    Value::Number(_) => "number",
                    Value::Bool(_) => "boolean",
                    Value::Array(_) => "array",
                    Value::Object(_) => "object",
                    Value::Null => "null",
                };
                
                if actual_type != expected_type {
                    errors.push(format!(
                        "Field {} should be type '{}' but got '{}'",
                        path, expected_type, actual_type
                    ));
                }
            }
            _ => {}  // Other types: just check presence
        }
    }
}

#[actix_web::test]
async fn test_product_api_contract() {
    let pool = setup_test_db().await;
    let app = create_test_app(pool).await;
    
    // กำหนด contract
    let contract = serde_json::json!({
        "data": "type:array",
        "meta": {
            "current_page": "type:number",
            "per_page": "type:number",
            "total_items": "type:number",
            "total_pages": "type:number",
            "has_next": "type:boolean",
            "has_prev": "type:boolean"
        }
    });
    
    let req = test::TestRequest::get()
        .uri("/api/v1/products")
        .to_request();
    
    let resp = test::call_service(&app, req).await;
    let body: serde_json::Value = test::read_body_json(resp).await;
    
    let api_contract = ApiContract {
        endpoint: "/api/v1/products".to_string(),
        method: "GET".to_string(),
        expected_response: contract,
    };
    
    match api_contract.verify_response(&body) {
        Ok(_) => println!("Contract test passed!"),
        Err(errors) => panic!("Contract test failed: {:?}", errors),
    }
}
```

## 7. E2E Testing

### 7.1 Playwright-style E2E Test

```rust
// tests/e2e/complete_flow_test.rs

#[actix_web::test]
async fn test_complete_shopping_flow() {
    let pool = setup_test_db().await;
    let app = create_full_test_app(pool).await;
    
    // 1. Register user
    let register_req = test::TestRequest::post()
        .uri("/api/v1/auth/register")
        .set_json(json!({
            "email": "e2e-test@example.com",
            "password": "Password123!",
            "name": "E2E Test User"
        }))
        .to_request();
    
    let register_resp = test::call_service(&app, register_req).await;
    assert_eq!(register_resp.status(), 201);
    
    // 2. Login
    let login_req = test::TestRequest::post()
        .uri("/api/v1/auth/login")
        .set_json(json!({
            "email": "e2e-test@example.com",
            "password": "Password123!"
        }))
        .to_request();
    
    let login_resp = test::call_service(&app, login_req).await;
    assert_eq!(login_resp.status(), 200);
    
    let login_body: serde_json::Value = test::read_body_json(login_resp).await;
    let token = login_body["access_token"].as_str().unwrap().to_string();
    
    // 3. Browse products
    let list_req = test::TestRequest::get()
        .uri("/api/v1/products?category=electronics")
        .insert_header(("Authorization", format!("Bearer {}", token)))
        .to_request();
    
    let list_resp = test::call_service(&app, list_req).await;
    assert_eq!(list_resp.status(), 200);
    
    let list_body: serde_json::Value = test::read_body_json(list_resp).await;
    let products = list_body["data"].as_array().unwrap();
    
    // 4. Create order
    if !products.is_empty() {
        let product_id = products[0]["id"].as_i64().unwrap();
        
        let order_req = test::TestRequest::post()
            .uri("/api/v1/orders")
            .insert_header(("Authorization", format!("Bearer {}", token)))
            .set_json(json!({
                "items": [{"product_id": product_id, "quantity": 1}]
            }))
            .to_request();
        
        let order_resp = test::call_service(&app, order_req).await;
        assert_eq!(order_resp.status(), 201);
        
        let order_body: serde_json::Value = test::read_body_json(order_resp).await;
        let order_id = order_body["id"].as_i64().unwrap();
        
        // 5. Check order status
        let status_req = test::TestRequest::get()
            .uri(&format!("/api/v1/orders/{}", order_id))
            .insert_header(("Authorization", format!("Bearer {}", token)))
            .to_request();
        
        let status_resp = test::call_service(&app, status_req).await;
        assert_eq!(status_resp.status(), 200);
        
        let status_body: serde_json::Value = test::read_body_json(status_resp).await;
        assert_eq!(status_body["status"], "pending");
    }
}
```

## 8. Practical: Complete Test Suite

### 8.1 Test Structure

```
tests/
├── common/
│   ├── mod.rs          # Shared utilities
│   ├── factories.rs    # Test data factories
│   └── auth.rs         # Auth helpers
├── unit/
│   ├── product_service_test.rs
│   ├── user_service_test.rs
│   └── validation_test.rs
├── integration/
│   ├── products_test.rs
│   ├── users_test.rs
│   └── orders_test.rs
├── contract/
│   └── api_contract_test.rs
└── e2e/
    └── shopping_flow_test.rs
```

### 8.2 Test Data Factories

```rust
// tests/common/factories.rs
use sqlx::PgPool;
use serde_json::json;

pub struct ProductFactory;

impl ProductFactory {
    pub async fn create(pool: &PgPool) -> i32 {
        Self::create_with(pool, |p| p).await
    }
    
    pub async fn create_with<F>(pool: &PgPool, modifier: F) -> i32
    where
        F: Fn(ProductBuilder) -> ProductBuilder,
    {
        let builder = modifier(ProductBuilder::default());
        
        let result = sqlx::query!(
            "INSERT INTO products (name, price, stock, category) VALUES ($1, $2, $3, $4) RETURNING id",
            builder.name,
            builder.price,
            builder.stock,
            builder.category
        )
        .fetch_one(pool)
        .await
        .unwrap();
        
        result.id
    }
}

#[derive(Default)]
pub struct ProductBuilder {
    pub name: String,
    pub price: f64,
    pub stock: i32,
    pub category: String,
}

impl ProductBuilder {
    pub fn name(mut self, name: &str) -> Self {
        self.name = name.to_string();
        self
    }
    
    pub fn price(mut self, price: f64) -> Self {
        self.price = price;
        self
    }
    
    pub fn out_of_stock(mut self) -> Self {
        self.stock = 0;
        self
    }
}

impl Default for ProductBuilder {
    fn default() -> Self {
        ProductBuilder {
            name: format!("Test Product {}", uuid::Uuid::new_v4()),
            price: 99.99,
            stock: 10,
            category: "test".to_string(),
        }
    }
}
```

### 8.3 Parallel Test Execution

```toml
# .cargo/config.toml
[test]
# Run tests in parallel
# แต่ database tests ต้องระวัง concurrent access
```

```bash
# Run tests แบบ parallel
cargo test -- --test-threads=4

# Run เฉพาะ unit tests (เร็ว)
cargo test --lib

# Run integration tests
cargo test --test '*'

# Run ด้วย output
cargo test -- --nocapture

# Run เฉพาะ test name ที่ match
cargo test test_product

# Generate coverage report
cargo tarpaulin --all-features --workspace --out Html
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Unit tests** - test business logic ที่แยกจาก I/O
2. **Integration tests** - test handlers กับ database
3. **testcontainers** - test กับ real database
4. **Mocking** - mock external services
5. **Coverage** - วัด code coverage ด้วย tarpaulin
6. **Contract testing** - ตรวจสอบ API contracts
7. **E2E testing** - test complete user flows
8. **Test organization** - โครงสร้าง test suite ที่ดี

---

[⬅️ Part 078: API Documentation with OpenAPI](../part_078/README.md) | [➡️ Part 080: Security Hardening](../part_080/README.md)
