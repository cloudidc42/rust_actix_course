# Part 090: Project: Real-time Collaboration 🤝

## 🎯 เป้าหมายของ Part นี้

- สร้าง real-time collaboration service
- Conflict resolution strategies
- Presence awareness
- Document versioning
- Operational transformation (OT) concepts

---

## 1. Project Overview

```
collaboration_service/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── models/
│   │   ├── document.rs
│   │   └── operation.rs
│   ├── ot/
│   │   └── transform.rs
│   ├── presence/
│   │   └── manager.rs
│   ├── handlers/
│   │   ├── documents.rs
│   │   └── ws.rs
│   └── state.rs
```

---

## 2. Cargo.toml

```toml
[package]
name = "collaboration_service"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = "4"
actix-ws = "0.3"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4", "serde"] }
chrono = { version = "0.4", features = ["serde"] }
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"] }
dashmap = "5"
futures = "0.3"
```

---

## 3. Operation Types (Operational Transformation)

```rust
// src/models/operation.rs
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum TextOperation {
    Insert { position: usize, text: String },
    Delete { position: usize, length: usize },
    Retain { length: usize },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Operation {
    pub id: Uuid,
    pub document_id: Uuid,
    pub user_id: Uuid,
    pub revision: u64,
    pub ops: Vec<TextOperation>,
    pub created_at: DateTime<Utc>,
}

impl Operation {
    pub fn new(document_id: Uuid, user_id: Uuid, revision: u64, ops: Vec<TextOperation>) -> Self {
        Self {
            id: Uuid::new_v4(),
            document_id,
            user_id,
            revision,
            ops,
            created_at: Utc::now(),
        }
    }
}
```

---

## 4. Operational Transformation Engine

```rust
// src/ot/transform.rs
use crate::models::operation::TextOperation;

pub fn apply_operation(content: &str, ops: &[TextOperation]) -> String {
    let mut result = content.to_string();
    let mut offset: i64 = 0;

    for op in ops {
        match op {
            TextOperation::Insert { position, text } => {
                let pos = (*position as i64 + offset).max(0) as usize;
                let pos = pos.min(result.len());
                result.insert_str(pos, text);
                offset += text.len() as i64;
            }
            TextOperation::Delete { position, length } => {
                let pos = (*position as i64 + offset).max(0) as usize;
                let pos = pos.min(result.len());
                let end = (pos + length).min(result.len());
                result.drain(pos..end);
                offset -= *length as i64;
            }
            TextOperation::Retain { .. } => {}
        }
    }

    result
}

// Transform operation A against operation B (concurrent operations)
pub fn transform(op_a: &[TextOperation], op_b: &[TextOperation]) -> Vec<TextOperation> {
    let mut transformed = Vec::new();
    let mut offset: i64 = 0;

    for a in op_a {
        match a {
            TextOperation::Insert { position, text } => {
                let new_pos = (*position as i64 + calculate_offset(op_b, *position)) as usize;
                transformed.push(TextOperation::Insert {
                    position: new_pos,
                    text: text.clone(),
                });
                offset += text.len() as i64;
            }
            TextOperation::Delete { position, length } => {
                let new_pos = (*position as i64 + offset) as usize;
                transformed.push(TextOperation::Delete {
                    position: new_pos,
                    length: *length,
                });
                offset -= *length as i64;
            }
            TextOperation::Retain { length } => {
                transformed.push(TextOperation::Retain { length: *length });
            }
        }
    }

    transformed
}

fn calculate_offset(ops: &[TextOperation], position: usize) -> i64 {
    let mut offset: i64 = 0;
    let mut current_pos = 0usize;

    for op in ops {
        match op {
            TextOperation::Insert { position: pos, text } => {
                if *pos <= position {
                    offset += text.len() as i64;
                }
                current_pos += text.len();
            }
            TextOperation::Delete { position: pos, length } => {
                if *pos < position {
                    let deleted = (*length).min(position - *pos);
                    offset -= deleted as i64;
                }
            }
            TextOperation::Retain { length } => {
                current_pos += length;
            }
        }
    }

    offset
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_insert() {
        let content = "Hello World";
        let ops = vec![TextOperation::Insert {
            position: 5,
            text: " Beautiful".to_string(),
        }];
        assert_eq!(apply_operation(content, &ops), "Hello Beautiful World");
    }

    #[test]
    fn test_delete() {
        let content = "Hello World";
        let ops = vec![TextOperation::Delete { position: 5, length: 6 }];
        assert_eq!(apply_operation(content, &ops), "Hello");
    }
}
```

---

## 5. Document Model

```rust
// src/models/document.rs
use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use uuid::Uuid;
use chrono::{DateTime, Utc};

#[derive(Debug, Clone, Serialize, FromRow)]
pub struct Document {
    pub id: Uuid,
    pub title: String,
    pub content: String,
    pub revision: i64,
    pub owner_id: Uuid,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}

#[derive(Debug, Deserialize)]
pub struct CreateDocumentRequest {
    pub title: String,
    pub content: Option<String>,
}
```

---

## 6. Presence Manager

```rust
// src/presence/manager.rs
use dashmap::DashMap;
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use uuid::Uuid;
use chrono::{DateTime, Utc};
use actix_ws::Session;

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct CursorPosition {
    pub line: u32,
    pub column: u32,
}

#[derive(Debug, Clone, Serialize)]
pub struct UserPresence {
    pub user_id: Uuid,
    pub username: String,
    pub cursor: Option<CursorPosition>,
    pub last_seen: DateTime<Utc>,
}

pub struct PresenceManager {
    // document_id -> user_id -> (presence, session)
    presence: DashMap<Uuid, DashMap<Uuid, (UserPresence, Session)>>,
}

impl PresenceManager {
    pub fn new() -> Arc<Self> {
        Arc::new(Self {
            presence: DashMap::new(),
        })
    }

    pub fn user_join(&self, doc_id: Uuid, user_id: Uuid, username: &str, session: Session) {
        let doc_presence = self.presence.entry(doc_id).or_insert_with(DashMap::new);
        doc_presence.insert(user_id, (
            UserPresence {
                user_id,
                username: username.to_string(),
                cursor: None,
                last_seen: Utc::now(),
            },
            session,
        ));
    }

    pub fn user_leave(&self, doc_id: Uuid, user_id: Uuid) {
        if let Some(doc_presence) = self.presence.get(&doc_id) {
            doc_presence.remove(&user_id);
        }
    }

    pub fn update_cursor(&self, doc_id: Uuid, user_id: Uuid, cursor: CursorPosition) {
        if let Some(doc_presence) = self.presence.get(&doc_id) {
            if let Some(mut entry) = doc_presence.get_mut(&user_id) {
                entry.0.cursor = Some(cursor);
                entry.0.last_seen = Utc::now();
            }
        }
    }

    pub fn get_users(&self, doc_id: Uuid) -> Vec<UserPresence> {
        self.presence
            .get(&doc_id)
            .map(|doc| doc.iter().map(|e| e.0.clone()).collect())
            .unwrap_or_default()
    }

    pub async fn broadcast(&self, doc_id: Uuid, message: &str, skip_user: Option<Uuid>) {
        if let Some(doc_presence) = self.presence.get(&doc_id) {
            for entry in doc_presence.iter() {
                if Some(*entry.key()) == skip_user {
                    continue;
                }
                let _ = entry.value().1.clone().text(message.to_string()).await;
            }
        }
    }
}
```

---

## 7. WebSocket Handler

```rust
// src/handlers/ws.rs
use actix_web::{get, web, HttpRequest, HttpResponse, Error};
use actix_ws::Message;
use futures::StreamExt;
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use std::sync::Arc;

use crate::models::operation::{Operation, TextOperation};
use crate::ot::transform::{apply_operation, transform};
use crate::presence::manager::{CursorPosition, PresenceManager};
use crate::state::AppState;

#[derive(Deserialize)]
#[serde(tag = "type")]
enum ClientMessage {
    Operation {
        document_id: Uuid,
        revision: u64,
        ops: Vec<TextOperation>,
    },
    CursorMove {
        document_id: Uuid,
        line: u32,
        column: u32,
    },
    Subscribe {
        document_id: Uuid,
    },
}

#[derive(Serialize)]
#[serde(tag = "type")]
enum ServerMessage {
    Operation {
        operation: Operation,
        content: String,
    },
    Presence {
        users: Vec<serde_json::Value>,
    },
    Error {
        message: String,
    },
    Ack {
        operation_id: Uuid,
    },
}

#[derive(Deserialize)]
struct WsQuery {
    user_id: Uuid,
    username: String,
}

#[get("/ws/document")]
pub async fn document_ws(
    req: HttpRequest,
    body: web::Payload,
    query: web::Query<WsQuery>,
    state: web::Data<Arc<AppState>>,
) -> Result<HttpResponse, Error> {
    let (response, session, mut msg_stream) = actix_ws::handle(&req, body)?;

    let user_id = query.user_id;
    let username = query.username.clone();
    let presence = state.presence.clone();
    let state = state.clone();
    let mut subscribed_doc: Option<Uuid> = None;

    actix_web::rt::spawn(async move {
        while let Some(Ok(msg)) = msg_stream.next().await {
            match msg {
                Message::Text(text) => {
                    let client_msg: ClientMessage = match serde_json::from_str(&text) {
                        Ok(m) => m,
                        Err(e) => {
                            let err = serde_json::to_string(&ServerMessage::Error {
                                message: format!("Invalid message: {}", e),
                            }).unwrap();
                            let _ = session.clone().text(err).await;
                            continue;
                        }
                    };

                    match client_msg {
                        ClientMessage::Subscribe { document_id } => {
                            subscribed_doc = Some(document_id);
                            presence.user_join(document_id, user_id, &username, session.clone());

                            // Send current users
                            let users = presence.get_users(document_id);
                            let msg = serde_json::to_string(&ServerMessage::Presence {
                                users: users.iter().map(|u| serde_json::json!({
                                    "user_id": u.user_id,
                                    "username": u.username,
                                    "cursor": u.cursor,
                                })).collect(),
                            }).unwrap();
                            let _ = session.clone().text(msg).await;
                        },
                        ClientMessage::Operation { document_id, revision, ops } => {
                            // Apply operation to document
                            let op = Operation::new(document_id, user_id, revision, ops.clone());
                            let op_id = op.id;

                            // Apply ops to content (simplified - real impl would use DB)
                            let content = apply_operation("", &ops);

                            // Broadcast to other users
                            let broadcast_msg = serde_json::to_string(&ServerMessage::Operation {
                                operation: op,
                                content: content.clone(),
                            }).unwrap();
                            presence.broadcast(document_id, &broadcast_msg, Some(user_id)).await;

                            // Ack to sender
                            let ack = serde_json::to_string(&ServerMessage::Ack {
                                operation_id: op_id,
                            }).unwrap();
                            let _ = session.clone().text(ack).await;
                        },
                        ClientMessage::CursorMove { document_id, line, column } => {
                            presence.update_cursor(document_id, user_id, CursorPosition { line, column });

                            // Broadcast cursor position
                            let cursor_msg = serde_json::to_string(&serde_json::json!({
                                "type": "CursorUpdate",
                                "user_id": user_id,
                                "username": username,
                                "line": line,
                                "column": column,
                            })).unwrap();
                            presence.broadcast(document_id, &cursor_msg, Some(user_id)).await;
                        }
                    }
                },
                Message::Ping(bytes) => {
                    let _ = session.clone().pong(&bytes).await;
                },
                Message::Close(_) => break,
                _ => {}
            }
        }

        // Cleanup on disconnect
        if let Some(doc_id) = subscribed_doc {
            presence.user_leave(doc_id, user_id);
            let leave_msg = serde_json::to_string(&serde_json::json!({
                "type": "UserLeft",
                "user_id": user_id,
            })).unwrap();
            presence.broadcast(doc_id, &leave_msg, None).await;
        }
    });

    Ok(response)
}
```

---

## 8. App State และ Main

```rust
// src/state.rs
use std::sync::Arc;
use sqlx::PgPool;
use crate::presence::manager::PresenceManager;

pub struct AppState {
    pub db: PgPool,
    pub presence: Arc<PresenceManager>,
}

impl AppState {
    pub fn new(db: PgPool) -> Arc<Self> {
        Arc::new(Self {
            db,
            presence: PresenceManager::new(),
        })
    }
}
```

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware::Logger};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;

mod models;
mod ot;
mod presence;
mod handlers;
mod state;

use state::AppState;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("info"));
    dotenv::dotenv().ok();

    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");

    let pool = PgPoolOptions::new()
        .max_connections(10)
        .connect(&database_url)
        .await
        .expect("Failed to connect to database");

    let app_state = AppState::new(pool);

    HttpServer::new(move || {
        App::new()
            .wrap(Logger::default())
            .app_data(web::Data::new(app_state.clone()))
            .service(handlers::documents::create_document)
            .service(handlers::documents::get_document)
            .service(handlers::ws::document_ws)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 9. ทดสอบ Collaboration

```javascript
// browser client example
const ws1 = new WebSocket('ws://localhost:8080/ws/document?user_id=UUID1&username=Alice');
const ws2 = new WebSocket('ws://localhost:8080/ws/document?user_id=UUID2&username=Bob');

// Subscribe to document
ws1.send(JSON.stringify({ type: 'Subscribe', document_id: 'DOC_UUID' }));
ws2.send(JSON.stringify({ type: 'Subscribe', document_id: 'DOC_UUID' }));

// Alice inserts text
ws1.send(JSON.stringify({
  type: 'Operation',
  document_id: 'DOC_UUID',
  revision: 1,
  ops: [{ type: 'Insert', position: 0, text: 'Hello' }]
}));

// Bob moves cursor
ws2.send(JSON.stringify({
  type: 'CursorMove',
  document_id: 'DOC_UUID',
  line: 1,
  column: 5
}));
```

---

## 10. สรุปสิ่งที่เรียนรู้

✅ Operational Transformation (OT) concepts  
✅ Conflict resolution strategies  
✅ Presence awareness (cursors, online users)  
✅ WebSocket-based collaboration  
✅ Document versioning  
✅ Broadcasting to room members  
✅ Real-time cursor tracking  

---

*[← Part 089: Project: API Gateway](../part_089/README.md) | [Part 091: Advanced Async Patterns →](../part_091/README.md)*
