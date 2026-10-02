# Part 058: gRPC with Tonic 🔗

## 🎯 เป้าหมายของ Part นี้

- Protocol Buffers (.proto files)
- tonic crate
- Server implementation
- Client implementation
- Streaming RPCs
- Metadata (headers)
- TLS สำหรับ gRPC
- Protobuf vs JSON comparison
- สร้าง User Service gRPC

---

## 1. Setup

```toml
# Cargo.toml
[package]
name = "grpc-user-service"
version = "0.1.0"
edition = "2021"

[dependencies]
tonic = { version = "0.11", features = ["tls"] }
prost = "0.12"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
serde = { version = "1", features = ["derive"] }
uuid = { version = "1", features = ["v4"] }
chrono = "0.4"
thiserror = "1"
anyhow = "1"

[build-dependencies]
tonic-build = "0.11"
```

---

## 2. Protocol Buffers Definition

```proto
// proto/user_service.proto
syntax = "proto3";

package user;

// User message
message User {
  string id = 1;
  string email = 2;
  string name = 3;
  string role = 4;
  bool active = 5;
  string created_at = 6;
  string updated_at = 7;
}

// Request/Response messages

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  User user = 1;
}

message CreateUserRequest {
  string email = 1;
  string name = 2;
  string password = 3;
  string role = 4;
}

message CreateUserResponse {
  User user = 1;
}

message UpdateUserRequest {
  string user_id = 1;
  optional string name = 2;
  optional string email = 3;
  optional bool active = 4;
}

message UpdateUserResponse {
  User user = 1;
}

message DeleteUserRequest {
  string user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 per_page = 2;
  string role_filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total = 2;
  int32 page = 3;
}

message SearchUsersRequest {
  string query = 1;
}

// Streaming
message UserEvent {
  string event_type = 1;  // "created", "updated", "deleted"
  User user = 2;
  string timestamp = 3;
}

// Service definition
service UserService {
  // Unary RPC
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
  rpc UpdateUser(UpdateUserRequest) returns (UpdateUserResponse);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);

  // Server streaming
  rpc WatchUsers(SearchUsersRequest) returns (stream UserEvent);

  // Client streaming (bulk import)
  rpc BulkCreateUsers(stream CreateUserRequest) returns (ListUsersResponse);

  // Bidirectional streaming
  rpc UserChat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string from_user_id = 1;
  string message = 2;
  string timestamp = 3;
}
```

---

## 3. Build Script

```rust
// build.rs
fn main() -> Result<(), Box<dyn std::error::Error>> {
    tonic_build::configure()
        .build_server(true)
        .build_client(true)
        .compile(
            &["proto/user_service.proto"],
            &["proto/"],
        )?;

    Ok(())
}
```

---

## 4. Server Implementation

```rust
// src/server.rs
use tonic::{transport::Server, Request, Response, Status};
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};
use uuid::Uuid;
use chrono::Utc;

// Include generated code
pub mod user_proto {
    tonic::include_proto!("user");
}

use user_proto::{
    user_service_server::{UserService, UserServiceServer},
    *,
};

#[derive(Debug, Default, Clone)]
pub struct UserRecord {
    pub id: String,
    pub email: String,
    pub name: String,
    pub role: String,
    pub active: bool,
    pub created_at: String,
    pub updated_at: String,
}

impl From<UserRecord> for User {
    fn from(r: UserRecord) -> Self {
        User {
            id: r.id,
            email: r.email,
            name: r.name,
            role: r.role,
            active: r.active,
            created_at: r.created_at,
            updated_at: r.updated_at,
        }
    }
}

pub struct UserServiceImpl {
    users: Arc<RwLock<Vec<UserRecord>>>,
    event_tx: broadcast::Sender<UserEvent>,
}

impl UserServiceImpl {
    pub fn new() -> Self {
        let (tx, _) = broadcast::channel(100);
        Self {
            users: Arc::new(RwLock::new(Vec::new())),
            event_tx: tx,
        }
    }
}

#[tonic::async_trait]
impl UserService for UserServiceImpl {
    // Unary: Get user by ID
    async fn get_user(
        &self,
        request: Request<GetUserRequest>,
    ) -> Result<Response<GetUserResponse>, Status> {
        let user_id = &request.into_inner().user_id;

        let users = self.users.read().await;
        let user = users
            .iter()
            .find(|u| &u.id == user_id)
            .cloned()
            .ok_or_else(|| Status::not_found(format!("User {} not found", user_id)))?;

        Ok(Response::new(GetUserResponse {
            user: Some(user.into()),
        }))
    }

    // Unary: Create user
    async fn create_user(
        &self,
        request: Request<CreateUserRequest>,
    ) -> Result<Response<CreateUserResponse>, Status> {
        let req = request.into_inner();

        // Validate
        if req.email.is_empty() {
            return Err(Status::invalid_argument("Email is required"));
        }

        let now = Utc::now().to_rfc3339();
        let user = UserRecord {
            id: Uuid::new_v4().to_string(),
            email: req.email,
            name: req.name,
            role: if req.role.is_empty() { "user".to_string() } else { req.role },
            active: true,
            created_at: now.clone(),
            updated_at: now,
        };

        // Broadcast event
        let _ = self.event_tx.send(UserEvent {
            event_type: "created".to_string(),
            user: Some(user.clone().into()),
            timestamp: Utc::now().to_rfc3339(),
        });

        self.users.write().await.push(user.clone());

        Ok(Response::new(CreateUserResponse {
            user: Some(user.into()),
        }))
    }

    // Unary: Update user
    async fn update_user(
        &self,
        request: Request<UpdateUserRequest>,
    ) -> Result<Response<UpdateUserResponse>, Status> {
        let req = request.into_inner();
        let mut users = self.users.write().await;

        let user = users
            .iter_mut()
            .find(|u| u.id == req.user_id)
            .ok_or_else(|| Status::not_found("User not found"))?;

        if let Some(name) = req.name {
            user.name = name;
        }
        if let Some(email) = req.email {
            user.email = email;
        }
        if let Some(active) = req.active {
            user.active = active;
        }
        user.updated_at = Utc::now().to_rfc3339();

        let updated = user.clone();

        // Broadcast event
        let _ = self.event_tx.send(UserEvent {
            event_type: "updated".to_string(),
            user: Some(updated.clone().into()),
            timestamp: Utc::now().to_rfc3339(),
        });

        Ok(Response::new(UpdateUserResponse {
            user: Some(updated.into()),
        }))
    }

    // Unary: Delete user
    async fn delete_user(
        &self,
        request: Request<DeleteUserRequest>,
    ) -> Result<Response<DeleteUserResponse>, Status> {
        let user_id = request.into_inner().user_id;
        let mut users = self.users.write().await;

        let pos = users
            .iter()
            .position(|u| u.id == user_id)
            .ok_or_else(|| Status::not_found("User not found"))?;

        let deleted = users.remove(pos);

        let _ = self.event_tx.send(UserEvent {
            event_type: "deleted".to_string(),
            user: Some(deleted.into()),
            timestamp: Utc::now().to_rfc3339(),
        });

        Ok(Response::new(DeleteUserResponse { success: true }))
    }

    // Unary: List users
    async fn list_users(
        &self,
        request: Request<ListUsersRequest>,
    ) -> Result<Response<ListUsersResponse>, Status> {
        let req = request.into_inner();
        let users = self.users.read().await;

        let filtered: Vec<User> = users
            .iter()
            .filter(|u| {
                req.role_filter.is_empty() || u.role == req.role_filter
            })
            .cloned()
            .map(Into::into)
            .collect();

        let total = filtered.len() as i32;
        let page = req.page.max(1);
        let per_page = req.per_page.max(1).min(100);
        let start = ((page - 1) * per_page) as usize;

        let paged: Vec<User> = filtered
            .into_iter()
            .skip(start)
            .take(per_page as usize)
            .collect();

        Ok(Response::new(ListUsersResponse {
            users: paged,
            total,
            page,
        }))
    }

    // Server Streaming: Watch user events
    type WatchUsersStream =
        tokio_stream::wrappers::BroadcastStream<UserEvent>;

    async fn watch_users(
        &self,
        request: Request<SearchUsersRequest>,
    ) -> Result<Response<Self::WatchUsersStream>, Status> {
        let query = request.into_inner().query;

        let rx = self.event_tx.subscribe();
        let stream = tokio_stream::wrappers::BroadcastStream::new(rx)
            .filter_map(move |r| {
                let query = query.clone();
                async move {
                    match r {
                        Ok(event) => {
                            if query.is_empty() {
                                Some(Ok(event))
                            } else if let Some(user) = &event.user {
                                if user.name.contains(&query)
                                    || user.email.contains(&query)
                                {
                                    Some(Ok(event))
                                } else {
                                    None
                                }
                            } else {
                                None
                            }
                        }
                        Err(_) => None,
                    }
                }
            });

        Ok(Response::new(stream))
    }

    // Client Streaming: Bulk create users
    async fn bulk_create_users(
        &self,
        request: Request<tonic::Streaming<CreateUserRequest>>,
    ) -> Result<Response<ListUsersResponse>, Status> {
        let mut stream = request.into_inner();
        let mut created: Vec<User> = Vec::new();

        while let Some(req) = stream.message().await? {
            let now = Utc::now().to_rfc3339();
            let user = UserRecord {
                id: Uuid::new_v4().to_string(),
                email: req.email,
                name: req.name,
                role: if req.role.is_empty() { "user".to_string() } else { req.role },
                active: true,
                created_at: now.clone(),
                updated_at: now,
            };

            self.users.write().await.push(user.clone());
            created.push(user.into());
        }

        let total = created.len() as i32;
        Ok(Response::new(ListUsersResponse {
            users: created,
            total,
            page: 1,
        }))
    }

    // Bidirectional Streaming: Chat
    type UserChatStream = tokio_stream::wrappers::ReceiverStream<Result<ChatMessage, Status>>;

    async fn user_chat(
        &self,
        request: Request<tonic::Streaming<ChatMessage>>,
    ) -> Result<Response<Self::UserChatStream>, Status> {
        let mut inbound = request.into_inner();
        let (tx, rx) = tokio::sync::mpsc::channel(100);

        tokio::spawn(async move {
            while let Some(msg) = inbound.message().await.unwrap_or(None) {
                let response = ChatMessage {
                    from_user_id: "server".to_string(),
                    message: format!("Echo: {}", msg.message),
                    timestamp: Utc::now().to_rfc3339(),
                };

                if tx.send(Ok(response)).await.is_err() {
                    break;
                }
            }
        });

        Ok(Response::new(tokio_stream::wrappers::ReceiverStream::new(rx)))
    }
}

// Start server
pub async fn start_server() -> anyhow::Result<()> {
    let addr = "0.0.0.0:50051".parse()?;
    let service = UserServiceImpl::new();

    println!("gRPC User Service listening on {}", addr);

    Server::builder()
        .add_service(UserServiceServer::new(service))
        .serve(addr)
        .await?;

    Ok(())
}
```

---

## 5. Client Implementation

```rust
// src/client.rs
use tonic::transport::Channel;
use user_proto::{user_service_client::UserServiceClient, *};

pub mod user_proto {
    tonic::include_proto!("user");
}

pub struct UserClient {
    client: UserServiceClient<Channel>,
}

impl UserClient {
    pub async fn connect(addr: &str) -> anyhow::Result<Self> {
        let client = UserServiceClient::connect(addr.to_string()).await?;
        Ok(Self { client })
    }

    pub async fn create_user(
        &mut self,
        email: &str,
        name: &str,
        password: &str,
    ) -> anyhow::Result<User> {
        let response = self
            .client
            .create_user(CreateUserRequest {
                email: email.to_string(),
                name: name.to_string(),
                password: password.to_string(),
                role: "user".to_string(),
            })
            .await?;

        Ok(response.into_inner().user.unwrap())
    }

    pub async fn get_user(&mut self, user_id: &str) -> anyhow::Result<User> {
        let response = self
            .client
            .get_user(GetUserRequest {
                user_id: user_id.to_string(),
            })
            .await?;

        Ok(response.into_inner().user.unwrap())
    }

    pub async fn list_users(&mut self, page: i32, per_page: i32) -> anyhow::Result<Vec<User>> {
        let response = self
            .client
            .list_users(ListUsersRequest {
                page,
                per_page,
                role_filter: String::new(),
            })
            .await?;

        Ok(response.into_inner().users)
    }

    // Server streaming
    pub async fn watch_users(&mut self) -> anyhow::Result<()> {
        let mut stream = self
            .client
            .watch_users(SearchUsersRequest {
                query: String::new(),
            })
            .await?
            .into_inner();

        println!("Watching for user events...");
        while let Some(event) = stream.message().await? {
            println!(
                "[Event] {} - {:?}",
                event.event_type,
                event.user.map(|u| u.name)
            );
        }

        Ok(())
    }

    // Client streaming (bulk import)
    pub async fn bulk_import(&mut self, users: Vec<(String, String)>) -> anyhow::Result<i32> {
        use tokio_stream::iter;

        let requests = iter(users.into_iter().map(|(email, name)| CreateUserRequest {
            email,
            name,
            password: "default_pass".to_string(),
            role: "user".to_string(),
        }));

        let response = self.client.bulk_create_users(requests).await?;
        Ok(response.into_inner().total)
    }
}

// Example usage
pub async fn run_client_demo() -> anyhow::Result<()> {
    let mut client = UserClient::connect("http://localhost:50051").await?;

    // Create user
    let user = client
        .create_user("alice@example.com", "Alice", "password123")
        .await?;
    println!("Created user: {} ({})", user.name, user.id);

    // Get user
    let fetched = client.get_user(&user.id).await?;
    println!("Fetched: {}", fetched.email);

    // List users
    let users = client.list_users(1, 10).await?;
    println!("Total users: {}", users.len());

    // Bulk import
    let bulk = vec![
        ("bob@example.com".to_string(), "Bob".to_string()),
        ("carol@example.com".to_string(), "Carol".to_string()),
    ];
    let count = client.bulk_import(bulk).await?;
    println!("Bulk imported: {} users", count);

    Ok(())
}
```

---

## 6. Metadata (Headers)

```rust
// src/metadata.rs
use tonic::{Request, Status, metadata::MetadataValue};

// เพิ่ม metadata ไปกับ request (client side)
pub fn add_auth_token(
    request: Request<impl std::fmt::Debug>,
    token: &str,
) -> Request<impl std::fmt::Debug> {
    let mut request = request;
    let token_value: MetadataValue<_> = format!("Bearer {}", token).parse().unwrap();
    request.metadata_mut().insert("authorization", token_value);
    request
}

// Interceptor สำหรับ authentication (server side)
pub fn auth_interceptor(
    request: Request<()>,
) -> Result<Request<()>, Status> {
    let token = match request.metadata().get("authorization") {
        Some(t) => t.to_str().map_err(|_| Status::unauthenticated("Invalid token format"))?,
        None => return Err(Status::unauthenticated("Missing authorization token")),
    };

    let token = token.strip_prefix("Bearer ").unwrap_or(token);

    // Validate token
    if token == "valid-token" {
        Ok(request)
    } else {
        Err(Status::unauthenticated("Invalid token"))
    }
}

// Middleware interceptor
use tonic::service::interceptor;

pub fn create_intercepted_service(service: UserServiceImpl) -> impl tonic::codegen::Service<...> {
    UserServiceServer::with_interceptor(service, auth_interceptor)
}
```

---

## 7. Protobuf vs JSON

| Feature | Protobuf | JSON |
|---------|----------|------|
| Size | ~3-10x เล็กกว่า | ใหญ่กว่า |
| Speed | ~5-10x เร็วกว่า | ช้ากว่า |
| Human-readable | ไม่ได้ | ได้ |
| Schema | ต้องมี .proto | ไม่จำเป็น |
| Type safety | เข้มแข็ง | อ่อนแอกว่า |
| Browser support | ต้องใช้ library | Built-in |

```rust
// Benchmark comparison (ตัวอย่าง)
use prost::Message;
use serde_json;

fn compare_serialization() {
    let user = User {
        id: "123".to_string(),
        name: "Alice".to_string(),
        email: "alice@example.com".to_string(),
        role: "admin".to_string(),
        active: true,
        created_at: "2024-01-01T00:00:00Z".to_string(),
        updated_at: "2024-01-01T00:00:00Z".to_string(),
    };

    // Protobuf
    let proto_bytes = user.encode_to_vec();
    println!("Protobuf size: {} bytes", proto_bytes.len()); // ~100 bytes

    // JSON
    let json_str = serde_json::to_string(&user).unwrap();
    println!("JSON size: {} bytes", json_str.len()); // ~200-300 bytes
}
```

---

## Main

```rust
// src/main.rs
use tokio;

mod server;
mod client;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let args: Vec<String> = std::env::args().collect();

    match args.get(1).map(|s| s.as_str()) {
        Some("server") => server::start_server().await?,
        Some("client") => client::run_client_demo().await?,
        _ => {
            println!("Usage: {} [server|client]", args[0]);
        }
    }

    Ok(())
}
```

---

## สรุป

✅ Protocol Buffers (.proto)  
✅ tonic server implementation  
✅ Client implementation  
✅ Unary RPC  
✅ Server streaming RPC  
✅ Client streaming RPC  
✅ Bidirectional streaming RPC  
✅ Metadata (auth tokens)  
✅ Protobuf vs JSON comparison  

### Exercise

1. เพิ่ม TLS สำหรับ production
2. สร้าง gRPC gateway สำหรับ HTTP/REST
3. เพิ่ม health check service
4. Implement load balancing

---

*[← Part 057: GraphQL with async-graphql](../part_057/README.md) | [Part 059: Message Queue with RabbitMQ →](../part_059/README.md)*
