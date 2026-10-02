# Part 044: Role-Based Access Control (RBAC) 🛡️

## 🎯 เป้าหมายของ Part นี้

- ออกแบบระบบ Role และ Permission
- สร้าง Role middleware
- ควบคุม access ระดับ route
- เก็บ roles ใน database
- Role hierarchy
- Resource-level permissions

---

## 1. RBAC Concepts

**RBAC (Role-Based Access Control)** คือการควบคุม access โดยกำหนด role ให้ user แล้วให้ role มี permissions

```
User → Role(s) → Permission(s) → Resource
```

**ตัวอย่าง:**
- Admin → [admin] → [create, read, update, delete] → [users, posts, settings]
- User → [user] → [create, read, update] → [own_posts] + [read] → [public_posts]
- Moderator → [moderator] → [read, update, delete] → [posts, comments]

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "rbac-system"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
jsonwebtoken = "9"
thiserror = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
futures = "0.3"
```

---

## 3. Role และ Permission Definitions

```rust
// src/rbac/roles.rs
use serde::{Deserialize, Serialize};
use std::collections::HashSet;

/// Roles ในระบบ
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Role {
    SuperAdmin,
    Admin,
    Moderator,
    User,
    Guest,
}

impl Role {
    pub fn as_str(&self) -> &str {
        match self {
            Role::SuperAdmin => "super_admin",
            Role::Admin => "admin",
            Role::Moderator => "moderator",
            Role::User => "user",
            Role::Guest => "guest",
        }
    }
    
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "super_admin" => Some(Role::SuperAdmin),
            "admin" => Some(Role::Admin),
            "moderator" => Some(Role::Moderator),
            "user" => Some(Role::User),
            "guest" => Some(Role::Guest),
            _ => None,
        }
    }
    
    /// Role hierarchy level (สูงกว่า = มี access มากกว่า)
    pub fn level(&self) -> u8 {
        match self {
            Role::SuperAdmin => 100,
            Role::Admin => 80,
            Role::Moderator => 60,
            Role::User => 40,
            Role::Guest => 10,
        }
    }
    
    /// ตรวจสอบว่า role นี้สูงกว่า (หรือเท่ากับ) role ที่กำหนดหรือไม่
    pub fn has_at_least(&self, required: &Role) -> bool {
        self.level() >= required.level()
    }
    
    /// Default permissions สำหรับแต่ละ role
    pub fn default_permissions(&self) -> HashSet<Permission> {
        match self {
            Role::SuperAdmin => {
                Permission::all()
            },
            Role::Admin => {
                let mut perms = HashSet::new();
                perms.insert(Permission::UserRead);
                perms.insert(Permission::UserCreate);
                perms.insert(Permission::UserUpdate);
                perms.insert(Permission::UserDelete);
                perms.insert(Permission::PostRead);
                perms.insert(Permission::PostCreate);
                perms.insert(Permission::PostUpdate);
                perms.insert(Permission::PostDelete);
                perms.insert(Permission::CommentRead);
                perms.insert(Permission::CommentCreate);
                perms.insert(Permission::CommentUpdate);
                perms.insert(Permission::CommentDelete);
                perms.insert(Permission::SettingsRead);
                perms.insert(Permission::SettingsUpdate);
                perms
            },
            Role::Moderator => {
                let mut perms = HashSet::new();
                perms.insert(Permission::UserRead);
                perms.insert(Permission::PostRead);
                perms.insert(Permission::PostUpdate);
                perms.insert(Permission::PostDelete);
                perms.insert(Permission::CommentRead);
                perms.insert(Permission::CommentUpdate);
                perms.insert(Permission::CommentDelete);
                perms
            },
            Role::User => {
                let mut perms = HashSet::new();
                perms.insert(Permission::PostRead);
                perms.insert(Permission::PostCreate);
                perms.insert(Permission::PostUpdate); // own posts only
                perms.insert(Permission::CommentRead);
                perms.insert(Permission::CommentCreate);
                perms.insert(Permission::CommentUpdate); // own comments only
                perms
            },
            Role::Guest => {
                let mut perms = HashSet::new();
                perms.insert(Permission::PostRead);
                perms.insert(Permission::CommentRead);
                perms
            },
        }
    }
}

/// Permissions ในระบบ
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Permission {
    // User permissions
    UserRead,
    UserCreate,
    UserUpdate,
    UserDelete,
    
    // Post permissions
    PostRead,
    PostCreate,
    PostUpdate,
    PostDelete,
    
    // Comment permissions
    CommentRead,
    CommentCreate,
    CommentUpdate,
    CommentDelete,
    
    // Settings permissions
    SettingsRead,
    SettingsUpdate,
    
    // Admin permissions
    AdminAccess,
    ManageRoles,
    ViewLogs,
    ManageSystem,
}

impl Permission {
    pub fn all() -> HashSet<Self> {
        let mut perms = HashSet::new();
        perms.insert(Permission::UserRead);
        perms.insert(Permission::UserCreate);
        perms.insert(Permission::UserUpdate);
        perms.insert(Permission::UserDelete);
        perms.insert(Permission::PostRead);
        perms.insert(Permission::PostCreate);
        perms.insert(Permission::PostUpdate);
        perms.insert(Permission::PostDelete);
        perms.insert(Permission::CommentRead);
        perms.insert(Permission::CommentCreate);
        perms.insert(Permission::CommentUpdate);
        perms.insert(Permission::CommentDelete);
        perms.insert(Permission::SettingsRead);
        perms.insert(Permission::SettingsUpdate);
        perms.insert(Permission::AdminAccess);
        perms.insert(Permission::ManageRoles);
        perms.insert(Permission::ViewLogs);
        perms.insert(Permission::ManageSystem);
        perms
    }
    
    pub fn as_str(&self) -> &str {
        match self {
            Permission::UserRead => "user:read",
            Permission::UserCreate => "user:create",
            Permission::UserUpdate => "user:update",
            Permission::UserDelete => "user:delete",
            Permission::PostRead => "post:read",
            Permission::PostCreate => "post:create",
            Permission::PostUpdate => "post:update",
            Permission::PostDelete => "post:delete",
            Permission::CommentRead => "comment:read",
            Permission::CommentCreate => "comment:create",
            Permission::CommentUpdate => "comment:update",
            Permission::CommentDelete => "comment:delete",
            Permission::SettingsRead => "settings:read",
            Permission::SettingsUpdate => "settings:update",
            Permission::AdminAccess => "admin:access",
            Permission::ManageRoles => "admin:manage_roles",
            Permission::ViewLogs => "admin:view_logs",
            Permission::ManageSystem => "admin:manage_system",
        }
    }
}
```

---

## 4. JWT Claims with Roles

```rust
// src/auth/claims.rs
use serde::{Deserialize, Serialize};
use jsonwebtoken::{decode, encode, DecodingKey, EncodingKey, Header, Validation};
use crate::rbac::roles::{Role, Permission};
use std::collections::HashSet;

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims {
    pub sub: String,         // user ID
    pub email: String,
    pub roles: Vec<String>,  // role names
    pub permissions: Vec<String>, // explicit permissions
    pub iat: i64,
    pub exp: i64,
}

impl Claims {
    pub fn new(
        user_id: &str,
        email: &str,
        roles: &[Role],
        extra_permissions: &[Permission],
    ) -> Self {
        let now = chrono::Utc::now();
        let exp = now + chrono::Duration::hours(24);
        
        // รวม permissions จากทุก role
        let mut all_permissions: HashSet<String> = roles.iter()
            .flat_map(|r| r.default_permissions())
            .map(|p| p.as_str().to_string())
            .collect();
        
        // เพิ่ม explicit permissions
        for perm in extra_permissions {
            all_permissions.insert(perm.as_str().to_string());
        }
        
        Self {
            sub: user_id.to_string(),
            email: email.to_string(),
            roles: roles.iter().map(|r| r.as_str().to_string()).collect(),
            permissions: all_permissions.into_iter().collect(),
            iat: now.timestamp(),
            exp: exp.timestamp(),
        }
    }
    
    pub fn has_role(&self, role: &Role) -> bool {
        self.roles.contains(&role.as_str().to_string())
    }
    
    pub fn has_permission(&self, permission: &Permission) -> bool {
        self.permissions.contains(&permission.as_str().to_string())
    }
    
    pub fn has_any_role(&self, roles: &[Role]) -> bool {
        roles.iter().any(|r| self.has_role(r))
    }
    
    pub fn has_all_permissions(&self, permissions: &[Permission]) -> bool {
        permissions.iter().all(|p| self.has_permission(p))
    }
    
    /// ตรวจสอบ minimum role level
    pub fn has_min_role_level(&self, min_role: &Role) -> bool {
        self.roles.iter()
            .filter_map(|r| Role::from_str(r))
            .any(|r| r.has_at_least(min_role))
    }
}

pub fn create_token(claims: &Claims, secret: &str) -> anyhow::Result<String> {
    let token = encode(
        &Header::default(),
        claims,
        &EncodingKey::from_secret(secret.as_bytes()),
    )?;
    Ok(token)
}

pub fn decode_token(token: &str, secret: &str) -> anyhow::Result<Claims> {
    let token_data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret.as_bytes()),
        &Validation::default(),
    )?;
    Ok(token_data.claims)
}
```

---

## 5. RBAC Middleware

```rust
// src/middleware/rbac.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpMessage, HttpResponse,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;
use crate::{auth::claims::{Claims, decode_token}, rbac::roles::{Role, Permission}};

/// Middleware สำหรับ require role
pub struct RequireRole {
    required_roles: Vec<Role>,
}

impl RequireRole {
    pub fn new(roles: Vec<Role>) -> Self {
        Self { required_roles: roles }
    }
    
    pub fn admin() -> Self {
        Self::new(vec![Role::Admin, Role::SuperAdmin])
    }
    
    pub fn moderator() -> Self {
        Self::new(vec![Role::Moderator, Role::Admin, Role::SuperAdmin])
    }
}

impl<S, B> Transform<S, ServiceRequest> for RequireRole
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = RequireRoleMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RequireRoleMiddleware {
            service: Rc::new(service),
            required_roles: self.required_roles.clone(),
        }))
    }
}

pub struct RequireRoleMiddleware<S> {
    service: Rc<S>,
    required_roles: Vec<Role>,
}

impl<S, B> Service<ServiceRequest> for RequireRoleMiddleware<S>
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
        let required_roles = self.required_roles.clone();
        
        Box::pin(async move {
            // ดึง JWT token จาก Authorization header
            let token = req
                .headers()
                .get("Authorization")
                .and_then(|h| h.to_str().ok())
                .and_then(|h| h.strip_prefix("Bearer "))
                .map(|t| t.to_string());
            
            let jwt_secret = std::env::var("JWT_SECRET").unwrap_or_default();
            
            let claims = match token {
                Some(t) => match decode_token(&t, &jwt_secret) {
                    Ok(c) => c,
                    Err(_) => {
                        let response = req.into_response(
                            HttpResponse::Unauthorized()
                                .json(serde_json::json!({"error": "Invalid token"}))
                                .map_into_boxed_body()
                        );
                        return Ok(response);
                    }
                },
                None => {
                    let response = req.into_response(
                        HttpResponse::Unauthorized()
                            .json(serde_json::json!({"error": "Authentication required"}))
                            .map_into_boxed_body()
                    );
                    return Ok(response);
                }
            };
            
            // ตรวจสอบ role
            let has_required_role = required_roles.iter().any(|r| claims.has_role(r));
            
            if !has_required_role {
                let response = req.into_response(
                    HttpResponse::Forbidden()
                        .json(serde_json::json!({
                            "error": "Insufficient permissions",
                            "required_roles": required_roles.iter()
                                .map(|r| r.as_str())
                                .collect::<Vec<_>>()
                        }))
                        .map_into_boxed_body()
                );
                return Ok(response);
            }
            
            // Insert claims ใน request extensions
            req.extensions_mut().insert(claims);
            
            service.call(req).await
        })
    }
}

/// Middleware สำหรับ require permission
pub struct RequirePermission {
    required_permissions: Vec<Permission>,
    require_all: bool,
}

impl RequirePermission {
    pub fn any(permissions: Vec<Permission>) -> Self {
        Self { required_permissions: permissions, require_all: false }
    }
    
    pub fn all(permissions: Vec<Permission>) -> Self {
        Self { required_permissions: permissions, require_all: true }
    }
}

impl<S, B> Transform<S, ServiceRequest> for RequirePermission
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = RequirePermissionMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(RequirePermissionMiddleware {
            service: Rc::new(service),
            required_permissions: self.required_permissions.clone(),
            require_all: self.require_all,
        }))
    }
}

pub struct RequirePermissionMiddleware<S> {
    service: Rc<S>,
    required_permissions: Vec<Permission>,
    require_all: bool,
}

impl<S, B> Service<ServiceRequest> for RequirePermissionMiddleware<S>
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
        let required_permissions = self.required_permissions.clone();
        let require_all = self.require_all;
        
        Box::pin(async move {
            let claims = req.extensions().get::<Claims>().cloned();
            
            let claims = match claims {
                Some(c) => c,
                None => {
                    let response = req.into_response(
                        HttpResponse::Unauthorized()
                            .json(serde_json::json!({"error": "Authentication required"}))
                            .map_into_boxed_body()
                    );
                    return Ok(response);
                }
            };
            
            let has_permission = if require_all {
                claims.has_all_permissions(&required_permissions)
            } else {
                required_permissions.iter().any(|p| claims.has_permission(p))
            };
            
            if !has_permission {
                let response = req.into_response(
                    HttpResponse::Forbidden()
                        .json(serde_json::json!({"error": "Insufficient permissions"}))
                        .map_into_boxed_body()
                );
                return Ok(response);
            }
            
            service.call(req).await
        })
    }
}
```

---

## 6. Database Schema

```sql
-- migrations/001_rbac.sql
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    level INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    resource VARCHAR(50),
    action VARCHAR(50),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE role_permissions (
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    permission_id UUID REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ,
    PRIMARY KEY (user_id, role_id)
);

-- Seed data
INSERT INTO roles (name, description, level) VALUES
    ('super_admin', 'Full system access', 100),
    ('admin', 'Administrative access', 80),
    ('moderator', 'Content moderation', 60),
    ('user', 'Regular user', 40),
    ('guest', 'Read-only access', 10);

INSERT INTO permissions (name, description, resource, action) VALUES
    ('user:read', 'Read user data', 'user', 'read'),
    ('user:create', 'Create users', 'user', 'create'),
    ('user:update', 'Update users', 'user', 'update'),
    ('user:delete', 'Delete users', 'user', 'delete'),
    ('post:read', 'Read posts', 'post', 'read'),
    ('post:create', 'Create posts', 'post', 'create'),
    ('post:update', 'Update posts', 'post', 'update'),
    ('post:delete', 'Delete posts', 'post', 'delete'),
    ('admin:access', 'Access admin panel', 'admin', 'access'),
    ('admin:manage_roles', 'Manage user roles', 'admin', 'manage_roles');
```

---

## 7. Handlers ต่างๆ

```rust
// src/handlers/admin.rs
use actix_web::{web, HttpRequest, HttpResponse};
use sqlx::PgPool;
use uuid::Uuid;
use crate::{
    auth::claims::Claims,
    rbac::roles::Role,
};

/// ดึง user claims จาก request extensions
fn get_claims(req: &HttpRequest) -> Option<Claims> {
    req.extensions().get::<Claims>().cloned()
}

/// List all users (Admin only)
pub async fn list_users(
    req: HttpRequest,
    pool: web::Data<PgPool>,
) -> actix_web::Result<HttpResponse> {
    let claims = get_claims(&req).ok_or_else(|| {
        actix_web::error::ErrorUnauthorized("Not authenticated")
    })?;
    
    // ตรวจสอบ permission เพิ่มเติม (double check)
    if !claims.has_min_role_level(&Role::Admin) {
        return Ok(HttpResponse::Forbidden().json(serde_json::json!({
            "error": "Admin access required"
        })));
    }
    
    let users = sqlx::query!(
        "SELECT id, email, created_at FROM users ORDER BY created_at DESC LIMIT 100"
    )
    .fetch_all(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Ok().json(
        users.iter().map(|u| serde_json::json!({
            "id": u.id,
            "email": u.email,
            "created_at": u.created_at,
        })).collect::<Vec<_>>()
    ))
}

/// Assign role to user (Super Admin only)
pub async fn assign_role(
    req: HttpRequest,
    pool: web::Data<PgPool>,
    path: web::Path<(Uuid, String)>,
) -> actix_web::Result<HttpResponse> {
    let claims = get_claims(&req).ok_or_else(|| {
        actix_web::error::ErrorUnauthorized("Not authenticated")
    })?;
    
    let (target_user_id, role_name) = path.into_inner();
    
    // ตรวจสอบว่า caller มี permission manage_roles
    if !claims.has_min_role_level(&Role::SuperAdmin) {
        return Ok(HttpResponse::Forbidden().json(serde_json::json!({
            "error": "Only super admin can assign roles"
        })));
    }
    
    // ดึง role ID
    let role = sqlx::query!(
        "SELECT id FROM roles WHERE name = $1",
        role_name
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?
    .ok_or_else(|| actix_web::error::ErrorNotFound("Role not found"))?;
    
    let granter_id: Uuid = claims.sub.parse()
        .map_err(|_| actix_web::error::ErrorBadRequest("Invalid user ID"))?;
    
    // Assign role
    sqlx::query!(
        r#"INSERT INTO user_roles (user_id, role_id, granted_by)
           VALUES ($1, $2, $3)
           ON CONFLICT (user_id, role_id) DO NOTHING"#,
        target_user_id,
        role.id,
        granter_id
    )
    .execute(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": format!("Role {} assigned to user {}", role_name, target_user_id)
    })))
}

/// Resource-level permission check
pub async fn update_post(
    req: HttpRequest,
    pool: web::Data<PgPool>,
    path: web::Path<Uuid>,
    body: web::Json<serde_json::Value>,
) -> actix_web::Result<HttpResponse> {
    let claims = get_claims(&req).ok_or_else(|| {
        actix_web::error::ErrorUnauthorized("Not authenticated")
    })?;
    
    let post_id = path.into_inner();
    let user_id: Uuid = claims.sub.parse()
        .map_err(|_| actix_web::error::ErrorBadRequest("Invalid user ID"))?;
    
    // ดึง post
    let post = sqlx::query!(
        "SELECT id, author_id, title FROM posts WHERE id = $1",
        post_id
    )
    .fetch_optional(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?
    .ok_or_else(|| actix_web::error::ErrorNotFound("Post not found"))?;
    
    // ตรวจสอบ: เจ้าของหรือ admin/moderator
    let can_edit = post.author_id == user_id
        || claims.has_min_role_level(&Role::Moderator);
    
    if !can_edit {
        return Ok(HttpResponse::Forbidden().json(serde_json::json!({
            "error": "You can only edit your own posts"
        })));
    }
    
    let title = body.get("title")
        .and_then(|v| v.as_str())
        .unwrap_or(&post.title);
    
    sqlx::query!(
        "UPDATE posts SET title = $1, updated_at = NOW() WHERE id = $2",
        title,
        post_id
    )
    .execute(pool.get_ref())
    .await
    .map_err(|e| actix_web::error::ErrorInternalServerError(e))?;
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "message": "Post updated"
    })))
}
```

---

## 8. Main Application

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware};
use sqlx::postgres::PgPoolOptions;
use dotenv::dotenv;

mod rbac {
    pub mod roles;
}
mod auth {
    pub mod claims;
}
mod middleware {
    pub mod rbac;
}
mod handlers {
    pub mod admin;
}

use middleware::rbac::{RequireRole, RequirePermission};
use rbac::roles::{Role, Permission};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let database_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");
    
    sqlx::migrate!("./migrations").run(&pool).await
        .expect("Failed to run migrations");
    
    HttpServer::new(move || {
        App::new()
            .app_data(web::Data::new(pool.clone()))
            .wrap(middleware::Logger::default())
            // Admin routes - require admin role
            .service(
                web::scope("/admin")
                    .wrap(RequireRole::admin())
                    .route("/users", web::get().to(handlers::admin::list_users))
                    .route("/users/{user_id}/roles/{role}", 
                           web::post().to(handlers::admin::assign_role))
            )
            // Post routes - require post:update permission
            .service(
                web::scope("/posts")
                    .route("/{id}", web::put().to(handlers::admin::update_post))
            )
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## 9. Testing RBAC

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::rbac::roles::{Role, Permission};
    use crate::auth::claims::Claims;

    #[test]
    fn test_role_hierarchy() {
        let super_admin = Role::SuperAdmin;
        let admin = Role::Admin;
        let user = Role::User;
        
        assert!(super_admin.has_at_least(&admin));
        assert!(admin.has_at_least(&user));
        assert!(!user.has_at_least(&admin));
    }
    
    #[test]
    fn test_claims_permissions() {
        let claims = Claims::new(
            "user-123",
            "test@example.com",
            &[Role::User],
            &[],
        );
        
        assert!(claims.has_role(&Role::User));
        assert!(!claims.has_role(&Role::Admin));
        assert!(claims.has_permission(&Permission::PostRead));
        assert!(!claims.has_permission(&Permission::AdminAccess));
    }
    
    #[test]
    fn test_admin_has_all_user_permissions() {
        let admin_perms = Role::Admin.default_permissions();
        let user_perms = Role::User.default_permissions();
        
        // Admin ต้องมี permissions ทุกอย่างที่ User มี
        for perm in &user_perms {
            assert!(admin_perms.contains(perm), 
                "Admin should have permission: {:?}", perm);
        }
    }
    
    #[test]
    fn test_min_role_level() {
        let claims = Claims::new(
            "admin-123",
            "admin@example.com",
            &[Role::Admin],
            &[],
        );
        
        assert!(claims.has_min_role_level(&Role::User));
        assert!(claims.has_min_role_level(&Role::Admin));
        assert!(!claims.has_min_role_level(&Role::SuperAdmin));
    }
}
```

---

## 10. สรุปสิ่งที่เรียนรู้

✅ Role definition (SuperAdmin, Admin, Moderator, User, Guest)  
✅ Permission system แบบ granular  
✅ Role hierarchy ด้วย level  
✅ JWT Claims พร้อม roles และ permissions  
✅ RequireRole middleware  
✅ RequirePermission middleware  
✅ Database-stored roles  
✅ Resource-level permission check  
✅ Role assignment โดย admin  

---

*[← Part 043: OAuth2 Integration](../part_043/README.md) | [Part 045: Rate Limiting →](../part_045/README.md)*
