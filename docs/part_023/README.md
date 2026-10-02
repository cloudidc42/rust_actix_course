# Part 023: JSON Request/Response ใน Actix-web

## สารบัญ
- [แนะนำ JSON ใน Rust](#แนะนำ-json-ใน-rust)
- [serde_json Value type](#serde_json-value-type)
- [Serialize/Deserialize derive](#serializedeserialize-derive)
- [Custom serialization](#custom-serialization)
- [Nested JSON structures](#nested-json-structures)
- [JSON arrays and objects](#json-arrays-and-objects)
- [Error handling for malformed JSON](#error-handling-for-malformed-json)
- [Pagination response pattern](#pagination-response-pattern)
- [Practical: Complete CRUD with JSON](#practical-complete-crud-with-json)

---

## แนะนำ JSON ใน Rust

JSON (JavaScript Object Notation) เป็นรูปแบบการแลกเปลี่ยนข้อมูลที่นิยมมากที่สุดใน Web API ใน Rust เราใช้ library ชื่อ `serde` และ `serde_json` ในการจัดการ JSON

### เพิ่ม Dependencies

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["full"] }
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4", "serde"] }
```

---

## serde_json Value type

`serde_json::Value` คือ enum ที่แทนค่า JSON ได้ทุกประเภท มีประโยชน์เมื่อเราไม่รู้โครงสร้างข้อมูลล่วงหน้า

```rust
use serde_json::{Value, json};

fn demonstrate_value_type() {
    // สร้าง JSON Value แบบต่างๆ
    let null_val: Value = Value::Null;
    let bool_val: Value = Value::Bool(true);
    let num_val: Value = Value::Number(42.into());
    let str_val: Value = Value::String("hello".to_string());
    
    // สร้างด้วย json! macro (วิธีที่นิยมกว่า)
    let object = json!({
        "name": "John Doe",
        "age": 30,
        "is_active": true,
        "score": 98.5,
        "tags": ["rust", "actix", "web"],
        "address": {
            "city": "Bangkok",
            "country": "Thailand"
        },
        "metadata": null
    });
    
    // การเข้าถึงข้อมูล
    if let Some(name) = object.get("name") {
        println!("Name: {}", name);
    }
    
    // การใช้ pointer notation
    if let Some(city) = object.pointer("/address/city") {
        println!("City: {}", city);
    }
    
    // ตรวจสอบประเภท
    match &object["age"] {
        Value::Number(n) => println!("Age is a number: {}", n),
        Value::String(s) => println!("Age is a string: {}", s),
        _ => println!("Age is something else"),
    }
    
    // แปลงเป็น String
    let json_string = object.to_string();
    println!("JSON: {}", json_string);
    
    // แปลงเป็น pretty-printed string
    let pretty = serde_json::to_string_pretty(&object).unwrap();
    println!("Pretty JSON:\n{}", pretty);
}

// ตัวอย่าง Handler ที่รับ dynamic JSON
use actix_web::{web, HttpResponse, Result};

async fn handle_dynamic_json(body: web::Json<Value>) -> Result<HttpResponse> {
    let data = body.into_inner();
    
    // ตรวจสอบว่ามี field ที่ต้องการหรือไม่
    let name = data.get("name")
        .and_then(|v| v.as_str())
        .unwrap_or("Unknown");
    
    let age = data.get("age")
        .and_then(|v| v.as_u64())
        .unwrap_or(0);
    
    // สร้าง response
    let response = json!({
        "status": "received",
        "name": name,
        "age": age,
        "message": format!("Hello, {}! You are {} years old.", name, age)
    });
    
    Ok(HttpResponse::Ok().json(response))
}
```

---

## Serialize/Deserialize derive

การใช้ `#[derive(Serialize, Deserialize)]` คือวิธีที่ง่ายที่สุดในการแปลง struct เป็น JSON และกลับมา

```rust
use serde::{Serialize, Deserialize};
use chrono::{DateTime, Utc};
use uuid::Uuid;

// Struct พื้นฐาน
#[derive(Debug, Serialize, Deserialize, Clone)]
struct User {
    id: Uuid,
    username: String,
    email: String,
    age: u32,
    is_active: bool,
    created_at: DateTime<Utc>,
    tags: Vec<String>,
    profile: Option<UserProfile>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct UserProfile {
    bio: Option<String>,
    avatar_url: Option<String>,
    website: Option<String>,
    location: Option<String>,
    social_links: Vec<SocialLink>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct SocialLink {
    platform: String,
    url: String,
}

impl User {
    fn new(username: String, email: String, age: u32) -> Self {
        User {
            id: Uuid::new_v4(),
            username,
            email,
            age,
            is_active: true,
            created_at: Utc::now(),
            tags: vec![],
            profile: None,
        }
    }
}

// Handler สำหรับสร้าง User
async fn create_user(body: web::Json<CreateUserRequest>) -> Result<HttpResponse> {
    let req = body.into_inner();
    
    let user = User::new(req.username, req.email, req.age);
    
    Ok(HttpResponse::Created().json(user))
}

#[derive(Debug, Deserialize)]
struct CreateUserRequest {
    username: String,
    email: String,
    age: u32,
    tags: Option<Vec<String>>,
}

// การ serialize/deserialize ด้วยตนเอง
fn serialize_example() {
    let user = User::new("john_doe".to_string(), "john@example.com".to_string(), 30);
    
    // Serialize เป็น JSON string
    let json = serde_json::to_string(&user).unwrap();
    println!("Serialized: {}", json);
    
    // Serialize เป็น pretty JSON
    let pretty_json = serde_json::to_string_pretty(&user).unwrap();
    println!("Pretty JSON:\n{}", pretty_json);
    
    // Deserialize จาก JSON string
    let json_str = r#"{
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "username": "jane_doe",
        "email": "jane@example.com",
        "age": 25,
        "is_active": true,
        "created_at": "2024-01-01T00:00:00Z",
        "tags": ["developer", "rust"],
        "profile": null
    }"#;
    
    match serde_json::from_str::<User>(json_str) {
        Ok(user) => println!("Deserialized: {:?}", user),
        Err(e) => println!("Error: {}", e),
    }
    
    // Serialize เป็น Vec<u8>
    let bytes = serde_json::to_vec(&user).unwrap();
    println!("Bytes length: {}", bytes.len());
    
    // Deserialize จาก bytes
    let user_from_bytes: User = serde_json::from_slice(&bytes).unwrap();
    println!("From bytes: {}", user_from_bytes.username);
}
```

---

## Custom serialization

`serde` มี attributes หลายอย่างที่ช่วยปรับแต่งการ serialize/deserialize

```rust
use serde::{Serialize, Deserialize, Serializer, Deserializer};
use serde::de::Error;

// #[serde(rename)] - เปลี่ยนชื่อ field ใน JSON
#[derive(Debug, Serialize, Deserialize)]
struct ProductDTO {
    #[serde(rename = "productId")]
    id: u64,
    
    #[serde(rename = "productName")]
    name: String,
    
    #[serde(rename = "unitPrice")]
    price: f64,
    
    #[serde(rename = "inStock")]
    is_in_stock: bool,
    
    #[serde(rename = "categoryId")]
    category_id: Option<u64>,
}

// #[serde(skip)] - ข้าม field นี้ทั้งในการ serialize และ deserialize
// #[serde(skip_serializing)] - ข้ามแค่ตอน serialize
// #[serde(skip_deserializing)] - ข้ามแค่ตอน deserialize
#[derive(Debug, Serialize, Deserialize)]
struct SecureUser {
    id: u64,
    username: String,
    email: String,
    
    #[serde(skip_serializing)]  // ไม่ส่ง password hash ออกไป
    password_hash: String,
    
    #[serde(skip)]  // ข้ามทั้งสองทาง (field นี้จะไม่ถูก serialize/deserialize)
    internal_cache: Option<String>,
    
    #[serde(skip_serializing_if = "Option::is_none")]  // ข้าม field ถ้าเป็น None
    bio: Option<String>,
    
    #[serde(skip_serializing_if = "Vec::is_empty")]  // ข้าม Vec ถ้าว่าง
    roles: Vec<String>,
}

impl Default for SecureUser {
    fn default() -> Self {
        SecureUser {
            id: 0,
            username: String::new(),
            email: String::new(),
            password_hash: String::new(),
            internal_cache: None,
            bio: None,
            roles: vec![],
        }
    }
}

// #[serde(default)] - ใช้ค่า default ถ้า field ไม่มีใน JSON
#[derive(Debug, Serialize, Deserialize)]
struct Config {
    host: String,
    
    #[serde(default = "default_port")]
    port: u16,
    
    #[serde(default = "default_max_connections")]
    max_connections: u32,
    
    #[serde(default)]  // ใช้ Default::default()
    debug_mode: bool,
    
    #[serde(default = "default_timeout")]
    timeout_seconds: u64,
}

fn default_port() -> u16 { 8080 }
fn default_max_connections() -> u32 { 100 }
fn default_timeout() -> u64 { 30 }

// Custom serialize/deserialize functions
#[derive(Debug, Serialize, Deserialize)]
struct Temperature {
    #[serde(serialize_with = "serialize_celsius", deserialize_with = "deserialize_celsius")]
    value: f64,
    unit: String,
}

fn serialize_celsius<S>(value: &f64, serializer: S) -> Result<S::Ok, S::Error>
where
    S: Serializer,
{
    // แปลงเป็น string พร้อม unit
    serializer.serialize_str(&format!("{:.2}°C", value))
}

fn deserialize_celsius<'de, D>(deserializer: D) -> Result<f64, D::Error>
where
    D: Deserializer<'de>,
{
    let s: String = String::deserialize(deserializer)?;
    // แยก numeric part ออกจาก string
    let num_str = s.trim_end_matches("°C").trim();
    num_str.parse::<f64>().map_err(D::Error::custom)
}

// #[serde(rename_all)] - เปลี่ยน naming convention ทั้ง struct
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]  // snake_case → camelCase
struct OrderItem {
    order_id: u64,
    product_id: u64,
    quantity: u32,
    unit_price: f64,
    total_price: f64,
    created_at: String,
}

// #[serde(rename_all = "SCREAMING_SNAKE_CASE")]
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "SCREAMING_SNAKE_CASE")]
enum Status {
    Active,
    Inactive,
    Pending,
    Suspended,
}

// Flattening nested structures
#[derive(Debug, Serialize, Deserialize)]
struct Address {
    street: String,
    city: String,
    country: String,
    postal_code: String,
}

#[derive(Debug, Serialize, Deserialize)]
struct Customer {
    id: u64,
    name: String,
    email: String,
    
    #[serde(flatten)]  // flatten ทำให้ fields ของ Address มาอยู่ใน Customer
    address: Address,
}

// ตัวอย่างการใช้งาน
fn custom_serde_examples() {
    // rename example
    let product = ProductDTO {
        id: 1,
        name: "Rust Book".to_string(),
        price: 29.99,
        is_in_stock: true,
        category_id: Some(5),
    };
    
    let json = serde_json::to_string_pretty(&product).unwrap();
    println!("Product JSON:\n{}", json);
    // Output จะแสดง "productId", "productName", etc.
    
    // flatten example
    let customer = Customer {
        id: 1,
        name: "สมชาย ใจดี".to_string(),
        email: "somchai@example.com".to_string(),
        address: Address {
            street: "123 ถนนสุขุมวิท".to_string(),
            city: "กรุงเทพฯ".to_string(),
            country: "Thailand".to_string(),
            postal_code: "10110".to_string(),
        },
    };
    
    let json = serde_json::to_string_pretty(&customer).unwrap();
    println!("Customer JSON:\n{}", json);
    // Address fields จะ flat อยู่ใน Customer object
}
```

---

## Nested JSON structures

การจัดการ JSON ซ้อนกันหลายชั้น

```rust
use serde::{Serialize, Deserialize};

// โครงสร้างข้อมูลซับซ้อน
#[derive(Debug, Serialize, Deserialize, Clone)]
struct Organization {
    id: u64,
    name: String,
    departments: Vec<Department>,
    headquarters: Location,
    metadata: OrgMetadata,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Department {
    id: u64,
    name: String,
    manager: Employee,
    employees: Vec<Employee>,
    budget: Budget,
    sub_departments: Vec<Department>,  // recursive structure!
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Employee {
    id: u64,
    name: String,
    position: String,
    contact: ContactInfo,
    skills: Vec<Skill>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct ContactInfo {
    email: String,
    phone: Option<String>,
    address: Option<Location>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Location {
    street: String,
    city: String,
    state: Option<String>,
    country: String,
    coordinates: Option<GeoCoordinates>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct GeoCoordinates {
    latitude: f64,
    longitude: f64,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Budget {
    amount: f64,
    currency: String,
    fiscal_year: u32,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Skill {
    name: String,
    level: SkillLevel,
    years_experience: Option<u32>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
#[serde(rename_all = "lowercase")]
enum SkillLevel {
    Beginner,
    Intermediate,
    Advanced,
    Expert,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct OrgMetadata {
    founded_year: u32,
    industry: String,
    employee_count: u32,
    website: Option<String>,
    social_media: std::collections::HashMap<String, String>,
}

// Handler ที่ทำงานกับ nested structures
use actix_web::{web, HttpResponse, Result};

async fn get_organization(
    path: web::Path<u64>,
) -> Result<HttpResponse> {
    let org_id = path.into_inner();
    
    // Simulate fetching from database
    let org = Organization {
        id: org_id,
        name: "TechCorp Thailand".to_string(),
        departments: vec![
            Department {
                id: 1,
                name: "Engineering".to_string(),
                manager: Employee {
                    id: 1,
                    name: "วิชัย เทคโนโลยี".to_string(),
                    position: "CTO".to_string(),
                    contact: ContactInfo {
                        email: "wichai@techcorp.th".to_string(),
                        phone: Some("+66-81-234-5678".to_string()),
                        address: None,
                    },
                    skills: vec![
                        Skill {
                            name: "Rust".to_string(),
                            level: SkillLevel::Expert,
                            years_experience: Some(5),
                        },
                        Skill {
                            name: "System Design".to_string(),
                            level: SkillLevel::Advanced,
                            years_experience: Some(10),
                        },
                    ],
                },
                employees: vec![],
                budget: Budget {
                    amount: 5_000_000.0,
                    currency: "THB".to_string(),
                    fiscal_year: 2024,
                },
                sub_departments: vec![
                    Department {
                        id: 11,
                        name: "Backend Team".to_string(),
                        manager: Employee {
                            id: 11,
                            name: "สมหมาย แบ็คเอนด์".to_string(),
                            position: "Backend Lead".to_string(),
                            contact: ContactInfo {
                                email: "sommai@techcorp.th".to_string(),
                                phone: None,
                                address: None,
                            },
                            skills: vec![],
                        },
                        employees: vec![],
                        budget: Budget {
                            amount: 2_000_000.0,
                            currency: "THB".to_string(),
                            fiscal_year: 2024,
                        },
                        sub_departments: vec![],
                    }
                ],
            }
        ],
        headquarters: Location {
            street: "99 ถนนรัชดาภิเษก".to_string(),
            city: "กรุงเทพฯ".to_string(),
            state: None,
            country: "Thailand".to_string(),
            coordinates: Some(GeoCoordinates {
                latitude: 13.7563,
                longitude: 100.5018,
            }),
        },
        metadata: OrgMetadata {
            founded_year: 2010,
            industry: "Technology".to_string(),
            employee_count: 500,
            website: Some("https://techcorp.th".to_string()),
            social_media: {
                let mut map = std::collections::HashMap::new();
                map.insert("twitter".to_string(), "@techcorp_th".to_string());
                map.insert("linkedin".to_string(), "techcorp-thailand".to_string());
                map
            },
        },
    };
    
    Ok(HttpResponse::Ok().json(org))
}

// Handler ที่รับและแก้ไข nested structure
async fn update_employee_skills(
    path: web::Path<(u64, u64)>,
    body: web::Json<Vec<Skill>>,
) -> Result<HttpResponse> {
    let (org_id, employee_id) = path.into_inner();
    let new_skills = body.into_inner();
    
    // ในการใช้งานจริงจะอัพเดทใน database
    let response = json!({
        "success": true,
        "org_id": org_id,
        "employee_id": employee_id,
        "updated_skills": new_skills,
        "updated_at": Utc::now()
    });
    
    Ok(HttpResponse::Ok().json(response))
}
```

---

## JSON arrays and objects

การจัดการ JSON arrays และ objects ที่หลากหลาย

```rust
use serde::{Serialize, Deserialize};
use std::collections::HashMap;

// Array of objects
#[derive(Debug, Serialize, Deserialize)]
struct BulkCreateRequest<T> {
    items: Vec<T>,
    options: Option<BulkOptions>,
}

#[derive(Debug, Serialize, Deserialize)]
struct BulkOptions {
    skip_duplicates: bool,
    return_created: bool,
    batch_size: Option<u32>,
}

#[derive(Debug, Serialize, Deserialize)]
struct BulkCreateResponse<T> {
    created: Vec<T>,
    failed: Vec<FailedItem>,
    total: u32,
    success_count: u32,
    failure_count: u32,
}

#[derive(Debug, Serialize, Deserialize)]
struct FailedItem {
    index: u32,
    error: String,
    data: serde_json::Value,
}

// Dynamic key-value pairs ด้วย HashMap
#[derive(Debug, Serialize, Deserialize)]
struct DynamicConfig {
    name: String,
    settings: HashMap<String, serde_json::Value>,
}

// Heterogeneous arrays (array ที่มีหลายประเภท)
#[derive(Debug, Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
enum EventData {
    UserSignup { user_id: u64, email: String },
    OrderPlaced { order_id: u64, total: f64 },
    PaymentReceived { payment_id: u64, amount: f64, method: String },
    SystemAlert { severity: String, message: String },
}

// Handlers
use actix_web::{web, HttpResponse, Result};
use chrono::Utc;

// รับ array ของ items
async fn bulk_create_users(
    body: web::Json<BulkCreateRequest<CreateUserRequest>>,
) -> Result<HttpResponse> {
    let request = body.into_inner();
    
    let mut created = vec![];
    let mut failed = vec![];
    
    for (index, item) in request.items.into_iter().enumerate() {
        // Validate each item
        if item.username.is_empty() {
            failed.push(FailedItem {
                index: index as u32,
                error: "Username cannot be empty".to_string(),
                data: serde_json::to_value(&item).unwrap_or(serde_json::Value::Null),
            });
            continue;
        }
        
        // Create user (simplified)
        created.push(User::new(item.username, item.email, item.age));
    }
    
    let total = (created.len() + failed.len()) as u32;
    let success_count = created.len() as u32;
    let failure_count = failed.len() as u32;
    
    let response = BulkCreateResponse {
        created,
        failed,
        total,
        success_count,
        failure_count,
    };
    
    if failure_count > 0 {
        Ok(HttpResponse::MultiStatus().json(response))
    } else {
        Ok(HttpResponse::Created().json(response))
    }
}

// รับ dynamic config
async fn update_config(
    body: web::Json<DynamicConfig>,
) -> Result<HttpResponse> {
    let config = body.into_inner();
    
    println!("Config name: {}", config.name);
    for (key, value) in &config.settings {
        println!("  {}: {:?}", key, value);
    }
    
    Ok(HttpResponse::Ok().json(json!({
        "status": "updated",
        "config_name": config.name,
        "settings_count": config.settings.len()
    })))
}

// ส่ง events array
async fn get_events() -> Result<HttpResponse> {
    let events = vec![
        EventData::UserSignup { user_id: 1, email: "test@example.com".to_string() },
        EventData::OrderPlaced { order_id: 100, total: 299.99 },
        EventData::PaymentReceived { 
            payment_id: 200, 
            amount: 299.99, 
            method: "credit_card".to_string() 
        },
    ];
    
    Ok(HttpResponse::Ok().json(events))
}
```

---

## Error handling for malformed JSON

การจัดการข้อผิดพลาดเมื่อได้รับ JSON ที่ไม่ถูกต้อง

```rust
use actix_web::{web, HttpResponse, Result, Error};
use actix_web::error::JsonPayloadError;
use serde::{Serialize, Deserialize};

// Custom error response
#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    message: String,
    details: Option<serde_json::Value>,
    status_code: u16,
}

// Custom JSON error handler
fn json_error_handler(err: JsonPayloadError, _req: &actix_web::HttpRequest) -> Error {
    let response = match &err {
        JsonPayloadError::ContentType => {
            HttpResponse::UnsupportedMediaType().json(ErrorResponse {
                error: "invalid_content_type".to_string(),
                message: "Content-Type must be application/json".to_string(),
                details: None,
                status_code: 415,
            })
        },
        JsonPayloadError::Deserialize(de_err) => {
            HttpResponse::BadRequest().json(ErrorResponse {
                error: "invalid_json".to_string(),
                message: format!("JSON parsing error: {}", de_err),
                details: Some(json!({
                    "line": de_err.line(),
                    "column": de_err.column(),
                    "category": format!("{:?}", de_err.classify())
                })),
                status_code: 400,
            })
        },
        JsonPayloadError::Overflow { limit } => {
            HttpResponse::PayloadTooLarge().json(ErrorResponse {
                error: "payload_too_large".to_string(),
                message: format!("Request body exceeds limit of {} bytes", limit),
                details: None,
                status_code: 413,
            })
        },
        _ => {
            HttpResponse::BadRequest().json(ErrorResponse {
                error: "bad_request".to_string(),
                message: err.to_string(),
                details: None,
                status_code: 400,
            })
        }
    };
    
    actix_web::error::InternalError::from_response(err, response).into()
}

// Struct สำหรับ validate
#[derive(Debug, Deserialize)]
struct CreateProductRequest {
    name: String,
    price: f64,
    quantity: u32,
    category: String,
}

// Handler พร้อม error handling
async fn create_product(
    body: web::Json<CreateProductRequest>,
) -> Result<HttpResponse> {
    let req = body.into_inner();
    
    // Manual validation
    let mut errors = vec![];
    
    if req.name.is_empty() {
        errors.push(json!({"field": "name", "message": "Name is required"}));
    }
    if req.name.len() > 100 {
        errors.push(json!({"field": "name", "message": "Name must be 100 characters or less"}));
    }
    if req.price <= 0.0 {
        errors.push(json!({"field": "price", "message": "Price must be greater than 0"}));
    }
    if req.price > 1_000_000.0 {
        errors.push(json!({"field": "price", "message": "Price is too high"}));
    }
    if req.category.is_empty() {
        errors.push(json!({"field": "category", "message": "Category is required"}));
    }
    
    if !errors.is_empty() {
        return Ok(HttpResponse::BadRequest().json(json!({
            "error": "validation_failed",
            "message": "Request validation failed",
            "details": errors
        })));
    }
    
    // สร้าง product
    let product = json!({
        "id": uuid::Uuid::new_v4(),
        "name": req.name,
        "price": req.price,
        "quantity": req.quantity,
        "category": req.category,
        "created_at": chrono::Utc::now()
    });
    
    Ok(HttpResponse::Created().json(product))
}

// การตั้งค่า JSON config ใน app
use actix_web::{web::JsonConfig, App, HttpServer};

async fn setup_app() {
    // การตั้งค่า JSON config
    let json_cfg = JsonConfig::default()
        .limit(1024 * 1024)  // 1MB limit
        .error_handler(json_error_handler);
    
    let _app = App::new()
        .app_data(json_cfg)
        .route("/products", web::post().to(create_product));
}
```

---

## Pagination response pattern

Pattern สำหรับ response ที่มีการแบ่งหน้า

```rust
use serde::{Serialize, Deserialize};

// Generic pagination response
#[derive(Debug, Serialize)]
struct PaginatedResponse<T> {
    data: Vec<T>,
    pagination: PaginationMeta,
}

#[derive(Debug, Serialize)]
struct PaginationMeta {
    total: u64,
    page: u64,
    per_page: u64,
    total_pages: u64,
    has_next: bool,
    has_prev: bool,
    next_page: Option<u64>,
    prev_page: Option<u64>,
}

impl PaginationMeta {
    fn new(total: u64, page: u64, per_page: u64) -> Self {
        let total_pages = if per_page > 0 { (total + per_page - 1) / per_page } else { 0 };
        let has_next = page < total_pages;
        let has_prev = page > 1;
        
        PaginationMeta {
            total,
            page,
            per_page,
            total_pages,
            has_next,
            has_prev,
            next_page: if has_next { Some(page + 1) } else { None },
            prev_page: if has_prev { Some(page - 1) } else { None },
        }
    }
}

impl<T> PaginatedResponse<T> {
    fn new(data: Vec<T>, total: u64, page: u64, per_page: u64) -> Self {
        PaginatedResponse {
            data,
            pagination: PaginationMeta::new(total, page, per_page),
        }
    }
}

// Query parameters สำหรับ pagination
#[derive(Debug, Deserialize)]
struct PaginationQuery {
    #[serde(default = "default_page")]
    page: u64,
    
    #[serde(default = "default_per_page")]
    per_page: u64,
    
    sort_by: Option<String>,
    sort_order: Option<SortOrder>,
    search: Option<String>,
}

fn default_page() -> u64 { 1 }
fn default_per_page() -> u64 { 20 }

#[derive(Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
enum SortOrder {
    Asc,
    Desc,
}

impl PaginationQuery {
    fn offset(&self) -> u64 {
        (self.page - 1) * self.per_page
    }
    
    fn limit(&self) -> u64 {
        self.per_page.min(100)  // max 100 per page
    }
}

// Cursor-based pagination
#[derive(Debug, Serialize, Deserialize)]
struct CursorPaginationQuery {
    cursor: Option<String>,
    limit: Option<u64>,
}

#[derive(Debug, Serialize)]
struct CursorPaginatedResponse<T> {
    data: Vec<T>,
    next_cursor: Option<String>,
    has_more: bool,
    count: usize,
}

// Product struct
#[derive(Debug, Serialize, Clone)]
struct Product {
    id: u64,
    name: String,
    price: f64,
    category: String,
    created_at: String,
}

// Handler ที่ใช้ offset pagination
use actix_web::{web, HttpResponse, Result};

async fn list_products(
    query: web::Query<PaginationQuery>,
) -> Result<HttpResponse> {
    let query = query.into_inner();
    
    // Simulate database query
    let all_products: Vec<Product> = (1..=100).map(|i| Product {
        id: i,
        name: format!("Product {}", i),
        price: (i as f64) * 10.5,
        category: if i % 3 == 0 { "Electronics" } else if i % 3 == 1 { "Clothing" } else { "Food" }.to_string(),
        created_at: "2024-01-01T00:00:00Z".to_string(),
    }).collect();
    
    // Filter by search
    let filtered: Vec<Product> = if let Some(ref search) = query.search {
        all_products.into_iter()
            .filter(|p| p.name.to_lowercase().contains(&search.to_lowercase()))
            .collect()
    } else {
        all_products
    };
    
    let total = filtered.len() as u64;
    let offset = query.offset() as usize;
    let limit = query.limit() as usize;
    
    // Get page slice
    let page_data: Vec<Product> = filtered
        .into_iter()
        .skip(offset)
        .take(limit)
        .collect();
    
    let response = PaginatedResponse::new(page_data, total, query.page, query.per_page);
    
    Ok(HttpResponse::Ok().json(response))
}

// Handler ที่ใช้ cursor pagination
async fn list_products_cursor(
    query: web::Query<CursorPaginationQuery>,
) -> Result<HttpResponse> {
    let query = query.into_inner();
    let limit = query.limit.unwrap_or(20).min(100) as usize;
    
    // Parse cursor (ปกติจะเป็น base64-encoded ID หรือ timestamp)
    let start_id = if let Some(cursor) = &query.cursor {
        // Decode cursor เป็น product ID
        use std::str::FromStr;
        u64::from_str(cursor).unwrap_or(0)
    } else {
        0
    };
    
    // Simulate fetching products after cursor
    let products: Vec<Product> = ((start_id + 1)..=(start_id + limit as u64 + 1))
        .take(limit + 1)  // fetch one extra to check if there's more
        .map(|i| Product {
            id: i,
            name: format!("Product {}", i),
            price: (i as f64) * 10.5,
            category: "Electronics".to_string(),
            created_at: "2024-01-01T00:00:00Z".to_string(),
        })
        .collect();
    
    let has_more = products.len() > limit;
    let mut data = products;
    if has_more {
        data.pop();  // remove the extra item
    }
    
    let next_cursor = if has_more {
        data.last().map(|p| p.id.to_string())
    } else {
        None
    };
    
    let count = data.len();
    let response = CursorPaginatedResponse {
        data,
        next_cursor,
        has_more,
        count,
    };
    
    Ok(HttpResponse::Ok().json(response))
}
```

---

## Practical: Complete CRUD with JSON

ตัวอย่าง CRUD API สมบูรณ์พร้อม JSON responses ที่ถูกต้อง

```rust
use actix_web::{web, App, HttpServer, HttpResponse, Result, middleware};
use serde::{Serialize, Deserialize};
use std::sync::{Arc, RwLock};
use std::collections::HashMap;
use uuid::Uuid;
use chrono::{DateTime, Utc};

// Models
#[derive(Debug, Serialize, Deserialize, Clone)]
struct Article {
    id: Uuid,
    title: String,
    content: String,
    author: String,
    tags: Vec<String>,
    published: bool,
    views: u64,
    created_at: DateTime<Utc>,
    updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
struct CreateArticleRequest {
    title: String,
    content: String,
    author: String,
    tags: Option<Vec<String>>,
    published: Option<bool>,
}

#[derive(Debug, Deserialize)]
struct UpdateArticleRequest {
    title: Option<String>,
    content: Option<String>,
    tags: Option<Vec<String>>,
    published: Option<bool>,
}

#[derive(Debug, Deserialize)]
struct ArticleQuery {
    page: Option<u64>,
    per_page: Option<u64>,
    author: Option<String>,
    tag: Option<String>,
    published: Option<bool>,
    search: Option<String>,
}

// App State
type ArticleStore = Arc<RwLock<HashMap<Uuid, Article>>>;

// Standard API Response wrapper
#[derive(Serialize)]
struct ApiResponse<T: Serialize> {
    success: bool,
    data: Option<T>,
    error: Option<String>,
    message: Option<String>,
    #[serde(skip_serializing_if = "Option::is_none")]
    meta: Option<serde_json::Value>,
}

impl<T: Serialize> ApiResponse<T> {
    fn success(data: T) -> Self {
        ApiResponse {
            success: true,
            data: Some(data),
            error: None,
            message: None,
            meta: None,
        }
    }
    
    fn success_with_meta(data: T, meta: serde_json::Value) -> Self {
        ApiResponse {
            success: true,
            data: Some(data),
            error: None,
            message: None,
            meta: Some(meta),
        }
    }
    
    fn error(message: &str) -> ApiResponse<()> {
        ApiResponse {
            success: false,
            data: None,
            error: Some(message.to_string()),
            message: None,
            meta: None,
        }
    }
}

// Handlers
async fn list_articles(
    store: web::Data<ArticleStore>,
    query: web::Query<ArticleQuery>,
) -> Result<HttpResponse> {
    let store = store.read().unwrap();
    let query = query.into_inner();
    
    let page = query.page.unwrap_or(1);
    let per_page = query.per_page.unwrap_or(10).min(50);
    
    let mut articles: Vec<Article> = store.values().cloned().collect();
    
    // Filter
    if let Some(author) = &query.author {
        articles.retain(|a| a.author.to_lowercase() == author.to_lowercase());
    }
    if let Some(tag) = &query.tag {
        articles.retain(|a| a.tags.iter().any(|t| t.to_lowercase() == tag.to_lowercase()));
    }
    if let Some(published) = query.published {
        articles.retain(|a| a.published == published);
    }
    if let Some(search) = &query.search {
        let search_lower = search.to_lowercase();
        articles.retain(|a| {
            a.title.to_lowercase().contains(&search_lower) ||
            a.content.to_lowercase().contains(&search_lower)
        });
    }
    
    // Sort by created_at desc
    articles.sort_by(|a, b| b.created_at.cmp(&a.created_at));
    
    let total = articles.len() as u64;
    let offset = ((page - 1) * per_page) as usize;
    let limit = per_page as usize;
    
    let page_data: Vec<Article> = articles.into_iter().skip(offset).take(limit).collect();
    
    let meta = json!({
        "pagination": {
            "total": total,
            "page": page,
            "per_page": per_page,
            "total_pages": (total + per_page - 1) / per_page,
            "has_next": page * per_page < total,
            "has_prev": page > 1
        }
    });
    
    Ok(HttpResponse::Ok().json(ApiResponse::success_with_meta(page_data, meta)))
}

async fn get_article(
    store: web::Data<ArticleStore>,
    path: web::Path<Uuid>,
) -> Result<HttpResponse> {
    let id = path.into_inner();
    let store = store.read().unwrap();
    
    match store.get(&id) {
        Some(article) => Ok(HttpResponse::Ok().json(ApiResponse::success(article))),
        None => Ok(HttpResponse::NotFound().json(ApiResponse::<()>::error("Article not found"))),
    }
}

async fn create_article(
    store: web::Data<ArticleStore>,
    body: web::Json<CreateArticleRequest>,
) -> Result<HttpResponse> {
    let req = body.into_inner();
    
    // Validation
    if req.title.trim().is_empty() {
        return Ok(HttpResponse::BadRequest().json(ApiResponse::<()>::error("Title is required")));
    }
    if req.content.trim().is_empty() {
        return Ok(HttpResponse::BadRequest().json(ApiResponse::<()>::error("Content is required")));
    }
    if req.title.len() > 200 {
        return Ok(HttpResponse::BadRequest().json(ApiResponse::<()>::error("Title must be 200 characters or less")));
    }
    
    let now = Utc::now();
    let article = Article {
        id: Uuid::new_v4(),
        title: req.title.trim().to_string(),
        content: req.content.trim().to_string(),
        author: req.author.trim().to_string(),
        tags: req.tags.unwrap_or_default(),
        published: req.published.unwrap_or(false),
        views: 0,
        created_at: now,
        updated_at: now,
    };
    
    let mut store = store.write().unwrap();
    let id = article.id;
    store.insert(id, article.clone());
    
    Ok(HttpResponse::Created().json(ApiResponse::success(article)))
}

async fn update_article(
    store: web::Data<ArticleStore>,
    path: web::Path<Uuid>,
    body: web::Json<UpdateArticleRequest>,
) -> Result<HttpResponse> {
    let id = path.into_inner();
    let req = body.into_inner();
    let mut store = store.write().unwrap();
    
    match store.get_mut(&id) {
        Some(article) => {
            if let Some(title) = req.title {
                if title.trim().is_empty() {
                    return Ok(HttpResponse::BadRequest().json(ApiResponse::<()>::error("Title cannot be empty")));
                }
                article.title = title.trim().to_string();
            }
            if let Some(content) = req.content {
                article.content = content.trim().to_string();
            }
            if let Some(tags) = req.tags {
                article.tags = tags;
            }
            if let Some(published) = req.published {
                article.published = published;
            }
            article.updated_at = Utc::now();
            
            Ok(HttpResponse::Ok().json(ApiResponse::success(article.clone())))
        },
        None => Ok(HttpResponse::NotFound().json(ApiResponse::<()>::error("Article not found"))),
    }
}

async fn delete_article(
    store: web::Data<ArticleStore>,
    path: web::Path<Uuid>,
) -> Result<HttpResponse> {
    let id = path.into_inner();
    let mut store = store.write().unwrap();
    
    match store.remove(&id) {
        Some(_) => Ok(HttpResponse::Ok().json(ApiResponse::success(json!({
            "deleted": true,
            "id": id
        })))),
        None => Ok(HttpResponse::NotFound().json(ApiResponse::<()>::error("Article not found"))),
    }
}

async fn increment_views(
    store: web::Data<ArticleStore>,
    path: web::Path<Uuid>,
) -> Result<HttpResponse> {
    let id = path.into_inner();
    let mut store = store.write().unwrap();
    
    match store.get_mut(&id) {
        Some(article) => {
            article.views += 1;
            Ok(HttpResponse::Ok().json(ApiResponse::success(json!({
                "views": article.views
            }))))
        },
        None => Ok(HttpResponse::NotFound().json(ApiResponse::<()>::error("Article not found"))),
    }
}

// App setup
#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let store: ArticleStore = Arc::new(RwLock::new(HashMap::new()));
    
    HttpServer::new(move || {
        let json_cfg = web::JsonConfig::default()
            .limit(1024 * 1024)  // 1MB
            .error_handler(json_error_handler);
        
        App::new()
            .app_data(web::Data::new(store.clone()))
            .app_data(json_cfg)
            .service(
                web::scope("/api/v1")
                    .route("/articles", web::get().to(list_articles))
                    .route("/articles", web::post().to(create_article))
                    .route("/articles/{id}", web::get().to(get_article))
                    .route("/articles/{id}", web::put().to(update_article))
                    .route("/articles/{id}", web::delete().to(delete_article))
                    .route("/articles/{id}/views", web::post().to(increment_views))
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **serde_json::Value** - การทำงานกับ dynamic JSON
2. **Serialize/Deserialize** - การแปลง struct ไป/จาก JSON
3. **Custom attributes** - `#[serde(rename)]`, `#[serde(skip)]`, `#[serde(default)]`, `#[serde(flatten)]`
4. **Nested structures** - การจัดการ JSON ซ้อนกันหลายชั้น
5. **Error handling** - การจัดการ JSON ที่ไม่ถูกต้อง
6. **Pagination** - รูปแบบ response สำหรับข้อมูลจำนวนมาก
7. **Complete CRUD** - ตัวอย่าง API ที่ใช้งานได้จริง

---

## การนำทาง

- [← Part 022: Database Integration](../part_022/README.md)
- [→ Part 024: Middleware in Actix-web](../part_024/README.md)
- [กลับหน้าหลัก](../../README.md)
