# Part 051: WebSockets Real-time 🔌

## 🎯 เป้าหมายของ Part นี้

- WebSocket ด้วย Actix-web
- Real-time chat application
- Broadcasting messages
- Room management
- Heartbeat / Ping-Pong

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
actix-ws = "0.3"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
```

---

## 2. Basic WebSocket

### 2.1 Simple Echo Server

```rust
use actix_web::{get, web, App, HttpRequest, HttpServer, HttpResponse, Error};
use actix_ws::Message;
use futures::StreamExt;

#[get("/ws")]
async fn websocket(
    req: HttpRequest,
    body: web::Payload,
) -> Result<HttpResponse, Error> {
    let (response, mut session, mut msg_stream) = actix_ws::handle(&req, body)?;

    // Handle messages in background
    actix_web::rt::spawn(async move {
        while let Some(Ok(msg)) = msg_stream.next().await {
            match msg {
                Message::Ping(bytes) => {
                    let _ = session.pong(&bytes).await;
                },
                Message::Text(text) => {
                    let echo = format!("Echo: {}", text);
                    let _ = session.text(echo).await;
                },
                Message::Binary(bytes) => {
                    let _ = session.binary(bytes).await;
                },
                Message::Close(reason) => {
                    let _ = session.close(reason).await;
                    break;
                },
                _ => {}
            }
        }
    });

    Ok(response)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .service(websocket)
            .service(actix_files::Files::new("/", "./static").index_file("index.html"))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### 2.2 WebSocket HTML Client

```html
<!-- static/index.html -->
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Demo</title>
    <style>
        body { font-family: Arial; max-width: 800px; margin: 50px auto; }
        #messages { height: 400px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; }
        .message { margin: 5px 0; padding: 5px; }
        .sent { background: #e3f2fd; text-align: right; }
        .received { background: #f1f8e9; }
        .system { color: #666; font-style: italic; }
        input[type=text] { width: 70%; padding: 8px; }
        button { padding: 8px 16px; }
    </style>
</head>
<body>
    <h1>🔌 WebSocket Demo</h1>
    <div id="messages"></div>
    <br>
    <input type="text" id="messageInput" placeholder="Type a message..." />
    <button onclick="sendMessage()">Send</button>
    <button onclick="connect()">Connect</button>
    <button onclick="disconnect()">Disconnect</button>
    <p>Status: <span id="status" style="color:red">Disconnected</span></p>

    <script>
        let ws = null;

        function addMessage(text, type = 'received') {
            const div = document.getElementById('messages');
            const msg = document.createElement('div');
            msg.className = `message ${type}`;
            msg.textContent = `${new Date().toLocaleTimeString()}: ${text}`;
            div.appendChild(msg);
            div.scrollTop = div.scrollHeight;
        }

        function connect() {
            if (ws) ws.close();
            ws = new WebSocket('ws://localhost:8080/ws');

            ws.onopen = () => {
                document.getElementById('status').textContent = 'Connected';
                document.getElementById('status').style.color = 'green';
                addMessage('Connected to server', 'system');
            };

            ws.onmessage = (event) => {
                addMessage(event.data, 'received');
            };

            ws.onclose = () => {
                document.getElementById('status').textContent = 'Disconnected';
                document.getElementById('status').style.color = 'red';
                addMessage('Disconnected from server', 'system');
            };

            ws.onerror = (error) => {
                addMessage('Error: ' + error, 'system');
            };
        }

        function disconnect() {
            if (ws) ws.close();
        }

        function sendMessage() {
            const input = document.getElementById('messageInput');
            if (ws && input.value) {
                ws.send(input.value);
                addMessage(input.value, 'sent');
                input.value = '';
            }
        }

        document.getElementById('messageInput').addEventListener('keypress', (e) => {
            if (e.key === 'Enter') sendMessage();
        });

        // Connect on load
        connect();
    </script>
</body>
</html>
```

---

## 3. Chat Application

### 3.1 Chat Room State

```rust
// src/chat.rs
use std::collections::HashMap;
use std::sync::{Arc, RwLock};
use actix_ws::Session;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", content = "data")]
pub enum ChatMessage {
    Join { user_id: String, username: String, room: String },
    Leave { user_id: String, username: String, room: String },
    Message { user_id: String, username: String, room: String, content: String },
    UserList { users: Vec<String> },
    Error { message: String },
}

#[derive(Clone)]
pub struct User {
    pub id: String,
    pub username: String,
    pub session: Session,
}

pub struct Room {
    pub id: String,
    pub name: String,
    pub users: HashMap<String, User>,
}

impl Room {
    pub fn new(name: &str) -> Self {
        Self {
            id: Uuid::new_v4().to_string(),
            name: name.to_string(),
            users: HashMap::new(),
        }
    }

    pub async fn broadcast(&self, message: &str, skip_user: Option<&str>) {
        for (user_id, user) in &self.users {
            if let Some(skip) = skip_user {
                if user_id == skip { continue; }
            }
            let _ = user.session.clone().text(message.to_string()).await;
        }
    }

    pub fn user_list(&self) -> Vec<String> {
        self.users.values().map(|u| u.username.clone()).collect()
    }
}

pub struct ChatState {
    pub rooms: RwLock<HashMap<String, Room>>,
}

impl ChatState {
    pub fn new() -> Arc<Self> {
        let state = Arc::new(Self {
            rooms: RwLock::new(HashMap::new()),
        });

        // Create default rooms
        {
            let mut rooms = state.rooms.write().unwrap();
            rooms.insert("general".to_string(), Room::new("general"));
            rooms.insert("rust".to_string(), Room::new("Rust Programming"));
            rooms.insert("actix".to_string(), Room::new("Actix-web"));
        }

        state
    }

    pub async fn user_join(
        state: &Arc<Self>,
        room_name: &str,
        user_id: &str,
        username: &str,
        session: Session,
    ) {
        let join_msg = ChatMessage::Join {
            user_id: user_id.to_string(),
            username: username.to_string(),
            room: room_name.to_string(),
        };
        let msg_json = serde_json::to_string(&join_msg).unwrap();

        let user_list;
        {
            let mut rooms = state.rooms.write().unwrap();
            let room = rooms.entry(room_name.to_string())
                .or_insert_with(|| Room::new(room_name));

            room.users.insert(user_id.to_string(), User {
                id: user_id.to_string(),
                username: username.to_string(),
                session,
            });

            user_list = room.user_list();

            // Broadcast join message
            for (uid, user) in &room.users {
                if uid != user_id {
                    let _ = user.session.clone().text(msg_json.clone()).await;
                }
            }
        }

        // Send user list to new user
        let list_msg = ChatMessage::UserList { users: user_list };
        let list_json = serde_json::to_string(&list_msg).unwrap();
        let rooms = state.rooms.read().unwrap();
        if let Some(room) = rooms.get(room_name) {
            if let Some(user) = room.users.get(user_id) {
                let _ = user.session.clone().text(list_json).await;
            }
        }
    }

    pub async fn user_leave(state: &Arc<Self>, room_name: &str, user_id: &str) {
        let leave_msg;
        {
            let mut rooms = state.rooms.write().unwrap();
            if let Some(room) = rooms.get_mut(room_name) {
                if let Some(user) = room.users.remove(user_id) {
                    leave_msg = Some(ChatMessage::Leave {
                        user_id: user.id.clone(),
                        username: user.username.clone(),
                        room: room_name.to_string(),
                    });
                } else {
                    leave_msg = None;
                }

                if let Some(msg) = &leave_msg {
                    let msg_json = serde_json::to_string(msg).unwrap();
                    for user in room.users.values() {
                        let _ = user.session.clone().text(msg_json.clone()).await;
                    }
                }
            }
        }
    }

    pub async fn send_message(
        state: &Arc<Self>,
        room_name: &str,
        user_id: &str,
        username: &str,
        content: &str,
    ) {
        let msg = ChatMessage::Message {
            user_id: user_id.to_string(),
            username: username.to_string(),
            room: room_name.to_string(),
            content: content.to_string(),
        };
        let msg_json = serde_json::to_string(&msg).unwrap();

        let rooms = state.rooms.read().unwrap();
        if let Some(room) = rooms.get(room_name) {
            for user in room.users.values() {
                let _ = user.session.clone().text(msg_json.clone()).await;
            }
        }
    }
}
```

### 3.2 WebSocket Handler

```rust
// src/handlers/ws.rs
use actix_web::{get, web, HttpRequest, HttpResponse, Error};
use actix_ws::Message;
use futures::StreamExt;
use std::sync::Arc;
use serde::Deserialize;
use uuid::Uuid;
use crate::chat::ChatState;

#[derive(Deserialize)]
struct WsQuery {
    username: String,
    room: Option<String>,
}

#[get("/ws/chat")]
async fn chat_ws(
    req: HttpRequest,
    body: web::Payload,
    query: web::Query<WsQuery>,
    state: web::Data<Arc<ChatState>>,
) -> Result<HttpResponse, Error> {
    let (response, session, mut msg_stream) = actix_ws::handle(&req, body)?;

    let user_id = Uuid::new_v4().to_string();
    let username = query.username.clone();
    let room = query.room.clone().unwrap_or_else(|| "general".to_string());
    let state = state.get_ref().clone();

    // User joins room
    ChatState::user_join(&state, &room, &user_id, &username, session.clone()).await;

    let uid = user_id.clone();
    let room_clone = room.clone();
    let state_clone = state.clone();
    let uname = username.clone();

    actix_web::rt::spawn(async move {
        while let Some(Ok(msg)) = msg_stream.next().await {
            match msg {
                Message::Text(text) => {
                    // Parse command or regular message
                    let content = text.trim();

                    if content.starts_with('/') {
                        // Command handling
                        handle_command(&state_clone, &uid, &uname, &room_clone, content).await;
                    } else {
                        ChatState::send_message(
                            &state_clone, &room_clone, &uid, &uname, content
                        ).await;
                    }
                },
                Message::Ping(bytes) => {
                    let _ = session.clone().pong(&bytes).await;
                },
                Message::Close(_) => {
                    break;
                },
                _ => {}
            }
        }

        // User leaves
        ChatState::user_leave(&state_clone, &room_clone, &uid).await;
        log::info!("User {} ({}) left room {}", uname, uid, room_clone);
    });

    Ok(response)
}

async fn handle_command(
    state: &Arc<ChatState>,
    user_id: &str,
    username: &str,
    current_room: &str,
    command: &str,
) {
    let parts: Vec<&str> = command.splitn(2, ' ').collect();
    match parts[0] {
        "/join" => {
            if let Some(new_room) = parts.get(1) {
                // Leave current, join new
                ChatState::user_leave(state, current_room, user_id).await;
                // Note: in real app you'd need to get the session again
                log::info!("{} joined room {}", username, new_room);
            }
        },
        "/list" => {
            log::info!("User {} requested user list", username);
        },
        _ => {}
    }
}
```

---

## 4. ทดสอบด้วย wscat

```bash
# ติดตั้ง wscat
npm install -g wscat

# Connect to WebSocket
wscat -c "ws://localhost:8080/ws/chat?username=Alice&room=general"

# ส่ง message
> Hello everyone!

# Another terminal
wscat -c "ws://localhost:8080/ws/chat?username=Bob&room=general"
> Hi Alice!
```

---

## 5. สรุปและ Exercises

### 5.1 สิ่งที่เรียนรู้

✅ WebSocket ด้วย actix-ws  
✅ Broadcasting messages  
✅ Room management  
✅ User join/leave events  
✅ Message types (JSON)  
✅ Command handling  

### 5.2 Exercise

**Exercise: Enhanced Chat**
1. Add authentication (JWT token ผ่าน query params)
2. Private messages (/dm username message)
3. Message history (เก็บใน Redis)
4. Typing indicators
5. Read receipts

---

*[← Part 050: Session Management](../part_050/README.md) | [Part 052: Server-Sent Events →](../part_052/README.md)*
