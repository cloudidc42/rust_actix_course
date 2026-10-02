# Part 041: JWT Authentication 🔐

## 🎯 เป้าหมายของ Part นี้

- Implement JWT authentication
- Login / Register endpoints
- JWT middleware
- Refresh tokens
- Role-based access

---

## 1. Setup

### 1.1 Cargo.toml

```toml
[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
jsonwebtoken = "9"
bcrypt = "0.15"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
dotenv = "0.15"
thiserror = "1"
log = "0.4"
env_logger = "0.11"
```

---

## 2. JWT Models

```rust
// src/auth/models.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,          // subject (user id)
    pub email: String,
    pub role: String,
    pub iat: i64,             // issued at
    pub exp: i64,             // expiration
    pub jti: String,          // JWT ID (for revocation)
    pub token_type: TokenType,
}

#[derive(Debug, Serialize, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum TokenType {
    Access,
    Refresh,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct TokenPair {
    pub access_token: String,
    pub refresh_token: String,
    pub token_type: String,
    pub expires_in: i64,  // seconds
}

#[derive(Debug, Serialize, Deserialize)]
pub struct LoginRequest {
    pub email: String,
    pub password: String,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct RefreshRequest {
    pub refresh_token: String,
}
```

---

## 3. JWT Service

```rust
// src/auth/jwt.rs
use jsonwebtoken::{decode, encode, Algorithm, DecodingKey, EncodingKey, Header, Validation};
use chrono::Utc;
use uuid::Uuid;
use crate::auth::models::{Claims, TokenPair, TokenType};

pub struct JwtConfig {
    pub secret: String,
    pub access_token_expire_secs: i64,
    pub refresh_token_expire_secs: i64,
}

impl JwtConfig {
    pub fn from_env() -> Self {
        Self {
            secret: std::env::var("JWT_SECRET")
                .unwrap_or_else(|_| "super_secret_key_change_in_production".to_string()),
            access_token_expire_secs: std::env::var("JWT_ACCESS_EXPIRE")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(3600),      // 1 hour
            refresh_token_expire_secs: std::env::var("JWT_REFRESH_EXPIRE")
                .ok()
                .and_then(|v| v.parse().ok())
                .unwrap_or(604800),    // 7 days
        }
    }
}

#[derive(Debug, thiserror::Error)]
pub enum JwtError {
    #[error("Token creation failed: {0}")]
    CreationError(#[from] jsonwebtoken::errors::Error),
    #[error("Token has expired")]
    Expired,
    #[error("Invalid token")]
    Invalid,
    #[error("Wrong token type: expected {expected}, got {got}")]
    WrongTokenType { expected: String, got: String },
}

pub fn create_token_pair(
    user_id: &str,
    email: &str,
    role: &str,
    config: &JwtConfig,
) -> Result<TokenPair, JwtError> {
    let access_token = create_token(
        user_id, email, role,
        TokenType::Access,
        config.access_token_expire_secs,
        &config.secret,
    )?;

    let refresh_token = create_token(
        user_id, email, role,
        TokenType::Refresh,
        config.refresh_token_expire_secs,
        &config.secret,
    )?;

    Ok(TokenPair {
        access_token,
        refresh_token,
        token_type: "Bearer".to_string(),
        expires_in: config.access_token_expire_secs,
    })
}

fn create_token(
    user_id: &str,
    email: &str,
    role: &str,
    token_type: TokenType,
    expire_secs: i64,
    secret: &str,
) -> Result<String, JwtError> {
    let now = Utc::now().timestamp();
    let claims = Claims {
        sub: user_id.to_string(),
        email: email.to_string(),
        role: role.to_string(),
        iat: now,
        exp: now + expire_secs,
        jti: Uuid::new_v4().to_string(),
        token_type,
    };

    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )
    .map_err(JwtError::CreationError)
}

pub fn verify_token(
    token: &str,
    secret: &str,
    expected_type: TokenType,
) -> Result<Claims, JwtError> {
    let mut validation = Validation::new(Algorithm::HS256);
    validation.validate_exp = true;

    let token_data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &validation,
    )
    .map_err(|e| match e.kind() {
        jsonwebtoken::errors::ErrorKind::ExpiredSignature => JwtError::Expired,
        _ => JwtError::Invalid,
    })?;

    if token_data.claims.token_type != expected_type {
        return Err(JwtError::WrongTokenType {
            expected: format!("{:?}", expected_type),
            got: format!("{:?}", token_data.claims.token_type),
        });
    }

    Ok(token_data.claims)
}
```

---

## 4. Authentication Middleware

```rust
// src/middleware/auth.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage,
};
use futures::future::{ready, LocalBoxFuture, Ready};
use std::rc::Rc;
use crate::auth::jwt::{verify_token, JwtConfig, JwtError};
use crate::auth::models::{Claims, TokenType};

pub struct AuthMiddleware {
    config: Rc<JwtConfig>,
    required_roles: Vec<String>,
}

impl AuthMiddleware {
    pub fn new(config: JwtConfig) -> Self {
        Self {
            config: Rc::new(config),
            required_roles: vec![],
        }
    }

    pub fn require_roles(mut self, roles: Vec<&str>) -> Self {
        self.required_roles = roles.iter().map(|s| s.to_string()).collect();
        self
    }
}

impl<S, B> Transform<S, ServiceRequest> for AuthMiddleware
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type InitError = ();
    type Transform = AuthMiddlewareService<S>;
    type Future = Ready<Result<Self::Transform, Self::InitError>>;

    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(AuthMiddlewareService {
            service: Rc::new(service),
            config: self.config.clone(),
            required_roles: self.required_roles.clone(),
        }))
    }
}

pub struct AuthMiddlewareService<S> {
    service: Rc<S>,
    config: Rc<JwtConfig>,
    required_roles: Vec<String>,
}

impl<S, B> Service<ServiceRequest> for AuthMiddlewareService<S>
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Future = LocalBoxFuture<'static, Result<Self::Response, Self::Error>>;

    forward_ready!(service);

    fn call(&self, req: ServiceRequest) -> Self::Future {
        let service = self.service.clone();
        let config = self.config.clone();
        let required_roles = self.required_roles.clone();

        Box::pin(async move {
            // Extract token from Authorization header
            let token = req
                .headers()
                .get("Authorization")
                .and_then(|v| v.to_str().ok())
                .and_then(|s| s.strip_prefix("Bearer "));

            let token = match token {
                Some(t) => t.to_string(),
                None => {
                    return Err(actix_web::error::ErrorUnauthorized(
                        serde_json::json!({"error": "Missing authorization token"})
                    ));
                }
            };

            // Verify token
            let claims = match verify_token(&token, &config.secret, TokenType::Access) {
                Ok(c) => c,
                Err(JwtError::Expired) => {
                    return Err(actix_web::error::ErrorUnauthorized(
                        serde_json::json!({"error": "Token has expired"})
                    ));
                },
                Err(_) => {
                    return Err(actix_web::error::ErrorUnauthorized(
                        serde_json::json!({"error": "Invalid token"})
                    ));
                }
            };

            // Check roles
            if !required_roles.is_empty() && !required_roles.contains(&claims.role) {
                return Err(actix_web::error::ErrorForbidden(
                    serde_json::json!({"error": "Insufficient permissions"})
                ));
            }

            // Attach claims to request extensions
            req.extensions_mut().insert(claims);

            service.call(req).await
        })
    }
}

// Helper to extract claims from request
pub fn get_claims(req: &actix_web::HttpRequest) -> Option<Claims> {
    req.extensions().get::<Claims>().cloned()
}
```

---

## 5. Auth Handlers

```rust
// src/handlers/auth.rs
use actix_web::{post, get, web, HttpRequest, HttpResponse};
use serde::{Deserialize, Serialize};
use crate::auth::jwt::{create_token_pair, verify_token, JwtConfig, JwtError};
use crate::auth::models::{LoginRequest, RefreshRequest, TokenType};
use crate::middleware::auth::get_claims;

#[post("/auth/register")]
async fn register(body: web::Json<RegisterRequest>) -> HttpResponse {
    // Validate
    if body.password.len() < 8 {
        return HttpResponse::BadRequest().json(serde_json::json!({
            "error": "Password must be at least 8 characters"
        }));
    }

    // Hash password
    let password_hash = bcrypt::hash(&body.password, bcrypt::DEFAULT_COST)
        .expect("Failed to hash password");

    // TODO: save to database
    // For now, return mock response
    HttpResponse::Created().json(serde_json::json!({
        "success": true,
        "message": "Registration successful",
        "user": {
            "id": "uuid-here",
            "email": body.email,
            "username": body.username
        }
    }))
}

#[post("/auth/login")]
async fn login(
    body: web::Json<LoginRequest>,
    jwt_config: web::Data<JwtConfig>,
) -> HttpResponse {
    // TODO: Look up user in database and verify password
    // Mock: accept admin@example.com / password
    if body.email != "admin@example.com" || body.password != "password" {
        return HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid credentials"
        }));
    }

    // Create tokens
    let user_id = "user-123";
    let role = "admin";

    match create_token_pair(user_id, &body.email, role, &jwt_config) {
        Ok(tokens) => HttpResponse::Ok().json(serde_json::json!({
            "success": true,
            "tokens": tokens,
            "user": {
                "id": user_id,
                "email": body.email,
                "role": role
            }
        })),
        Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({
            "error": e.to_string()
        }))
    }
}

#[post("/auth/refresh")]
async fn refresh_token(
    body: web::Json<RefreshRequest>,
    jwt_config: web::Data<JwtConfig>,
) -> HttpResponse {
    match verify_token(&body.refresh_token, &jwt_config.secret, TokenType::Refresh) {
        Ok(claims) => {
            match create_token_pair(&claims.sub, &claims.email, &claims.role, &jwt_config) {
                Ok(tokens) => HttpResponse::Ok().json(serde_json::json!({
                    "success": true,
                    "tokens": tokens
                })),
                Err(e) => HttpResponse::InternalServerError().json(serde_json::json!({
                    "error": e.to_string()
                }))
            }
        },
        Err(JwtError::Expired) => HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Refresh token has expired, please login again"
        })),
        Err(_) => HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Invalid refresh token"
        }))
    }
}

#[get("/auth/me")]
async fn get_me(req: HttpRequest) -> HttpResponse {
    match get_claims(&req) {
        Some(claims) => HttpResponse::Ok().json(serde_json::json!({
            "id": claims.sub,
            "email": claims.email,
            "role": claims.role
        })),
        None => HttpResponse::Unauthorized().json(serde_json::json!({
            "error": "Not authenticated"
        }))
    }
}

#[derive(Deserialize)]
struct RegisterRequest {
    username: String,
    email: String,
    password: String,
}

pub fn configure(cfg: &mut web::ServiceConfig) {
    cfg
        .service(register)
        .service(login)
        .service(refresh_token)
        .service(get_me);
}
```

---

## 6. Main.rs กับ Auth

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware::Logger, HttpResponse, get};
use crate::auth::jwt::JwtConfig;
use crate::middleware::auth::AuthMiddleware;

mod auth;
mod middleware;
mod handlers;

#[get("/api/admin/dashboard")]
async fn admin_dashboard(req: actix_web::HttpRequest) -> HttpResponse {
    let claims = middleware::auth::get_claims(&req).unwrap();
    HttpResponse::Ok().json(serde_json::json!({
        "message": "Admin dashboard",
        "admin": claims.email
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv::dotenv().ok();
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));

    let jwt_config = JwtConfig::from_env();

    HttpServer::new(move || {
        let jwt_config_data = web::Data::new(JwtConfig::from_env());

        App::new()
            .wrap(Logger::default())
            .app_data(jwt_config_data)
            // Public routes
            .service(
                web::scope("/api")
                    .configure(handlers::auth::configure)
            )
            // Protected routes (require authentication)
            .service(
                web::scope("/api/protected")
                    .wrap(AuthMiddleware::new(JwtConfig::from_env()))
                    .route("/profile", web::get().to(|| async {
                        HttpResponse::Ok().body("Your profile")
                    }))
            )
            // Admin routes (require admin role)
            .service(
                web::scope("/api/admin")
                    .wrap(
                        AuthMiddleware::new(JwtConfig::from_env())
                            .require_roles(vec!["admin"])
                    )
                    .service(admin_dashboard)
            )
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 7. ทดสอบ

```bash
# Register
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"johndoe","email":"john@example.com","password":"secret123"}'

# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}'

# Save token
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tokens']['access_token'])")

# Access protected route
curl http://localhost:8080/api/protected/profile \
  -H "Authorization: Bearer $TOKEN"

# Access admin route
curl http://localhost:8080/api/admin/dashboard \
  -H "Authorization: Bearer $TOKEN"

# Get current user
curl http://localhost:8080/api/auth/me \
  -H "Authorization: Bearer $TOKEN"

# Refresh token
REFRESH=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['tokens']['refresh_token'])")

curl -X POST http://localhost:8080/api/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refresh_token\":\"$REFRESH\"}"
```

---

## 8. สรุปและ Exercises

### 8.1 สิ่งที่เรียนรู้

✅ JWT token creation และ verification  
✅ Access token + Refresh token  
✅ Authentication middleware  
✅ Role-based access control  
✅ Token extraction จาก headers  
✅ Claims ใน request extensions  

### 8.2 Exercises

**Exercise: Complete Auth System**
1. Connect กับ database จริง
2. Password hashing ด้วย bcrypt
3. Token blacklist (revocation)
4. Email verification
5. Password reset flow

---

*[← Part 040: Full REST API](../part_040/README.md) | [Part 042: Password Hashing →](../part_042/README.md)*
