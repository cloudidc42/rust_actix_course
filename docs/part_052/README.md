# Part 052: Server-Sent Events (SSE) 📡

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ SSE vs WebSocket
- ใช้งาน SSE ด้วย actix-web-lab
- Event format (data, id, event, retry)
- Custom event types
- Heartbeat events
- Real-time notifications
- Browser EventSource API
- Reconnection handling
- สร้าง Live Notifications System

---

## 1. SSE vs WebSocket

SSE (Server-Sent Events) และ WebSocket เป็นสองวิธีหลักในการรับข้อมูล real-time จาก server

| Feature | SSE | WebSocket |
|---------|-----|-----------|
| Direction | Server → Client (one-way) | Bidirectional |
| Protocol | HTTP/HTTPS | ws:// wss:// |
| Reconnection | Auto | Manual |
| Browser support | Built-in EventSource | Built-in WebSocket |
| Load balancer | ง่ายกว่า (HTTP) | ยากกว่า (Upgrade header) |
| Use case | Notifications, feeds | Chat, gaming, real-time collab |

**เมื่อไหร่ควรใช้ SSE:**
- แสดง live feed (news, stock prices)
- Push notifications
- Progress tracking
- Live logs

---

## 2. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
actix-web-lab = "0.20"
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
futures-util = "0.3"
chrono = { version = "0.4", features = ["serde"] }
```

---

## 3. Basic SSE

### 3.1 Simple SSE Endpoint

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, HttpRequest, HttpResponse};
use actix_web_lab::sse::{self, Sse, ChannelStream};
use std::time::Duration;
use tokio::time::sleep;

async fn basic_sse() -> Sse<ChannelStream> {
    let (sender, stream) = sse::channel(10);

    tokio::spawn(async move {
        for i in 0..5 {
            sleep(Duration::from_secs(1)).await;

            let data = serde_json::json!({
                "count": i,
                "message": format!("Event #{}", i)
            });

            if sender
                .send(sse::Data::new(data.to_string()))
                .await
                .is_err()
            {
                break; // Client disconnected
            }
        }

        // ส่ง event สุดท้าย
        let _ = sender
            .send(sse::Data::new("done").event("complete"))
            .await;
    });

    Sse::from_channel_stream(stream)
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/events", web::get().to(basic_sse))
            .route("/", web::get().to(index))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}

async fn index() -> HttpResponse {
    HttpResponse::Ok()
        .content_type("text/html")
        .body(include_str!("../static/index.html"))
}
```

### 3.2 Event Format

SSE ใช้รูปแบบข้อความเป็น plain text:

```
data: Hello World\n\n
```

Fields ที่รองรับ:
- `data:` - ข้อมูล (required)
- `id:` - Event ID สำหรับ reconnection
- `event:` - Custom event type
- `retry:` - Reconnection interval (ms)

```rust
// src/event_format.rs

use actix_web_lab::sse;

// Data เดียว
let simple = sse::Data::new("Hello World");

// Data พร้อม event type
let typed = sse::Data::new("payload").event("user_joined");

// Data พร้อม id
let with_id = sse::Data::new_json(&my_struct)
    .unwrap()
    .id("event-123");

// Data พร้อม retry
let with_retry = sse::Data::new("reconnect in 3s")
    .retry(Duration::from_secs(3));
```

---

## 4. Custom Event Types

```rust
// src/events.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type", content = "payload")]
pub enum NotificationEvent {
    UserJoined { user_id: String, username: String },
    UserLeft { user_id: String },
    NewMessage { from: String, message: String, timestamp: String },
    SystemAlert { level: String, message: String },
    DataUpdate { resource: String, action: String, id: String },
}

impl NotificationEvent {
    pub fn event_type(&self) -> &'static str {
        match self {
            Self::UserJoined { .. } => "user_joined",
            Self::UserLeft { .. } => "user_left",
            Self::NewMessage { .. } => "new_message",
            Self::SystemAlert { .. } => "system_alert",
            Self::DataUpdate { .. } => "data_update",
        }
    }
}
```

```rust
// src/sse_handler.rs
use actix_web::{web, HttpResponse};
use actix_web_lab::sse::{self, Sse, ChannelStream};
use std::time::Duration;
use tokio::time::interval;
use crate::events::NotificationEvent;
use chrono::Utc;

pub async fn notification_sse(
    state: web::Data<AppState>,
) -> Sse<ChannelStream> {
    let (sender, stream) = sse::channel(100);

    // Subscribe ไปยัง notification channel
    let mut rx = state.notification_tx.subscribe();

    tokio::spawn(async move {
        let mut heartbeat = interval(Duration::from_secs(30));

        loop {
            tokio::select! {
                // รับ notification จาก channel
                Ok(event) = rx.recv() => {
                    let event_type = event.event_type();
                    let json = serde_json::to_string(&event).unwrap_or_default();

                    if sender
                        .send(sse::Data::new(json).event(event_type))
                        .await
                        .is_err()
                    {
                        break; // Client disconnected
                    }
                }

                // Heartbeat ทุก 30 วินาที
                _ = heartbeat.tick() => {
                    if sender
                        .send(sse::Data::new("ping").event("heartbeat"))
                        .await
                        .is_err()
                    {
                        break;
                    }
                }
            }
        }
    });

    Sse::from_channel_stream(stream)
        .with_keep_alive(Duration::from_secs(15))
}
```

---

## 5. Heartbeat Events

Heartbeat ใช้เพื่อ:
1. ตรวจสอบว่า connection ยังอยู่
2. ป้องกัน proxy timeout
3. บอก client ว่า server ยังทำงานอยู่

```rust
// src/heartbeat.rs
use actix_web_lab::sse;
use std::time::Duration;
use tokio::time::interval;

pub struct SseHeartbeat {
    sender: sse::Sender,
    interval: Duration,
}

impl SseHeartbeat {
    pub fn new(sender: sse::Sender, interval: Duration) -> Self {
        Self { sender, interval }
    }

    pub fn start(self) -> tokio::task::JoinHandle<()> {
        tokio::spawn(async move {
            let mut ticker = interval(self.interval);

            loop {
                ticker.tick().await;

                let timestamp = chrono::Utc::now().to_rfc3339();
                let payload = serde_json::json!({
                    "timestamp": timestamp,
                    "status": "alive"
                });

                if self
                    .sender
                    .send(
                        sse::Data::new(payload.to_string())
                            .event("heartbeat")
                            .id(uuid::Uuid::new_v4().to_string()),
                    )
                    .await
                    .is_err()
                {
                    break;
                }
            }
        })
    }
}
```

---

## 6. AppState and Broadcast

```rust
// src/state.rs
use tokio::sync::broadcast;
use crate::events::NotificationEvent;

#[derive(Clone)]
pub struct AppState {
    pub notification_tx: broadcast::Sender<NotificationEvent>,
}

impl AppState {
    pub fn new() -> Self {
        let (tx, _) = broadcast::channel(1000);
        Self {
            notification_tx: tx,
        }
    }

    pub fn broadcast(&self, event: NotificationEvent) {
        // ไม่ต้องสนใจ error ถ้าไม่มี subscribers
        let _ = self.notification_tx.send(event);
    }
}
```

---

## 7. SSE กับ Authentication

```rust
// src/auth_sse.rs
use actix_web::{web, HttpRequest, HttpResponse};
use actix_web_lab::sse::{self, Sse, ChannelStream};
use crate::{state::AppState, auth::validate_token};

pub async fn authenticated_sse(
    req: HttpRequest,
    state: web::Data<AppState>,
) -> Result<Sse<ChannelStream>, actix_web::Error> {
    // ตรวจสอบ token จาก query string หรือ header
    let token = req
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .or_else(|| {
            req.uri()
                .query()
                .and_then(|q| {
                    url::form_urlencoded::parse(q.as_bytes())
                        .find(|(k, _)| k == "token")
                        .map(|(_, v)| v)
                        .map(|v| v.into_owned())
                })
                .as_deref()
                .map(|s| s)
        });

    // validate token
    // ...ตรวจสอบ token จริงๆ...

    let (sender, stream) = sse::channel(100);
    let mut rx = state.notification_tx.subscribe();

    tokio::spawn(async move {
        while let Ok(event) = rx.recv().await {
            let json = serde_json::to_string(&event).unwrap_or_default();
            if sender.send(sse::Data::new(json)).await.is_err() {
                break;
            }
        }
    });

    Ok(Sse::from_channel_stream(stream))
}
```

---

## 8. Browser EventSource API

```html
<!-- static/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <title>SSE Notifications</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; }
        #notifications { list-style: none; padding: 0; }
        #notifications li {
            padding: 10px;
            margin: 5px 0;
            border-radius: 5px;
            background: #f0f0f0;
        }
        .user_joined { background: #d4edda !important; }
        .new_message { background: #cce5ff !important; }
        .system_alert { background: #fff3cd !important; }
        #status { font-weight: bold; }
        .connected { color: green; }
        .disconnected { color: red; }
        .reconnecting { color: orange; }
    </style>
</head>
<body>
    <h1>📡 Live Notifications</h1>
    <p>Status: <span id="status" class="disconnected">Disconnected</span></p>
    <ul id="notifications"></ul>

    <script>
        const statusEl = document.getElementById('status');
        const list = document.getElementById('notifications');
        let eventSource = null;
        let reconnectAttempts = 0;

        function connect() {
            statusEl.textContent = 'Connecting...';
            statusEl.className = 'reconnecting';

            // สร้าง EventSource พร้อม token
            eventSource = new EventSource('/api/notifications?token=my-token');

            // Event เมื่อ connected
            eventSource.onopen = () => {
                statusEl.textContent = 'Connected';
                statusEl.className = 'connected';
                reconnectAttempts = 0;
            };

            // Error handler
            eventSource.onerror = (e) => {
                statusEl.textContent = 'Reconnecting...';
                statusEl.className = 'reconnecting';
            };

            // Default message handler
            eventSource.onmessage = (e) => {
                addNotification('message', e.data);
            };

            // Custom event handlers
            eventSource.addEventListener('user_joined', (e) => {
                const data = JSON.parse(e.data);
                addNotification('user_joined', `👤 ${data.username} joined`);
            });

            eventSource.addEventListener('new_message', (e) => {
                const data = JSON.parse(e.data);
                addNotification('new_message', `💬 ${data.from}: ${data.message}`);
            });

            eventSource.addEventListener('system_alert', (e) => {
                const data = JSON.parse(e.data);
                addNotification('system_alert', `⚠️ [${data.level}] ${data.message}`);
            });

            eventSource.addEventListener('heartbeat', (e) => {
                console.log('Heartbeat received:', e.data);
            });

            eventSource.addEventListener('complete', () => {
                eventSource.close();
                statusEl.textContent = 'Stream completed';
            });
        }

        function addNotification(type, message) {
            const li = document.createElement('li');
            li.className = type;
            li.textContent = `[${new Date().toLocaleTimeString()}] ${message}`;
            list.prepend(li);

            // จำกัดจำนวน notifications
            if (list.children.length > 50) {
                list.removeChild(list.lastChild);
            }
        }

        function disconnect() {
            if (eventSource) {
                eventSource.close();
                statusEl.textContent = 'Disconnected';
                statusEl.className = 'disconnected';
            }
        }

        // เริ่ม connect
        connect();
    </script>
</body>
</html>
```

---

## 9. Reconnection Handling

Browser EventSource จะ reconnect อัตโนมัติ แต่เราควบคุมได้ด้วย `retry:` field

```rust
// src/reconnect.rs
use actix_web_lab::sse;
use std::time::Duration;

// ส่ง retry interval ให้ browser
pub async fn send_with_retry(
    sender: &sse::Sender,
    data: &str,
    event_id: &str,
) -> Result<(), ()> {
    sender
        .send(
            sse::Data::new(data)
                .id(event_id)
                .retry(Duration::from_millis(3000)), // reconnect ใน 3 วินาที
        )
        .await
        .map_err(|_| ())
}
```

### Last-Event-ID Header

เมื่อ client reconnect, browser จะส่ง `Last-Event-ID` header มาด้วย:

```rust
// src/resume_sse.rs
use actix_web::{web, HttpRequest};
use actix_web_lab::sse::{self, Sse, ChannelStream};

pub async fn resumable_sse(
    req: HttpRequest,
    state: web::Data<AppState>,
) -> Sse<ChannelStream> {
    // อ่าน last event ID สำหรับ resume
    let last_event_id = req
        .headers()
        .get("Last-Event-ID")
        .and_then(|v| v.to_str().ok())
        .map(|s| s.to_string());

    let (sender, stream) = sse::channel(100);

    tokio::spawn(async move {
        // ส่ง missed events ถ้ามี
        if let Some(last_id) = last_event_id {
            // TODO: ดึง events ที่ missed จาก database
            println!("Client reconnecting from event ID: {}", last_id);
            // ส่ง buffered events ก่อน...
        }

        // ส่ง events ปกติต่อไป
        let mut rx = state.notification_tx.subscribe();
        let mut event_counter: u64 = 0;

        while let Ok(event) = rx.recv().await {
            event_counter += 1;
            let json = serde_json::to_string(&event).unwrap_or_default();
            let event_id = format!("evt-{}", event_counter);

            if sender
                .send(sse::Data::new(json).id(event_id))
                .await
                .is_err()
            {
                break;
            }
        }
    });

    Sse::from_channel_stream(stream)
}
```

---

## 10. Practical: Live Notifications System

```rust
// src/main.rs - Complete notification system
use actix_web::{web, App, HttpServer, HttpRequest, HttpResponse, middleware};
use actix_web_lab::sse::{self, Sse, ChannelStream};
use serde::{Deserialize, Serialize};
use std::{sync::Arc, time::Duration};
use tokio::sync::{broadcast, RwLock};
use uuid::Uuid;
use chrono::Utc;

// Event types
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SseEvent {
    pub id: String,
    pub event_type: String,
    pub payload: serde_json::Value,
    pub timestamp: String,
}

impl SseEvent {
    pub fn new(event_type: &str, payload: serde_json::Value) -> Self {
        Self {
            id: Uuid::new_v4().to_string(),
            event_type: event_type.to_string(),
            payload,
            timestamp: Utc::now().to_rfc3339(),
        }
    }
}

// Connected client tracker
#[derive(Clone)]
pub struct ClientInfo {
    pub id: String,
    pub user_id: Option<String>,
    pub connected_at: String,
}

// App state
pub struct AppState {
    pub event_tx: broadcast::Sender<SseEvent>,
    pub clients: Arc<RwLock<Vec<ClientInfo>>>,
}

impl AppState {
    pub fn new() -> Self {
        let (tx, _) = broadcast::channel(1000);
        Self {
            event_tx: tx,
            clients: Arc::new(RwLock::new(Vec::new())),
        }
    }

    pub async fn add_client(&self, info: ClientInfo) {
        self.clients.write().await.push(info);
    }

    pub async fn remove_client(&self, client_id: &str) {
        self.clients.write().await.retain(|c| c.id != client_id);
    }

    pub fn broadcast(&self, event: SseEvent) {
        let _ = self.event_tx.send(event);
    }
}

// SSE endpoint
async fn sse_notifications(
    req: HttpRequest,
    state: web::Data<AppState>,
) -> Sse<ChannelStream> {
    let client_id = Uuid::new_v4().to_string();
    let client_info = ClientInfo {
        id: client_id.clone(),
        user_id: None, // จาก auth token
        connected_at: Utc::now().to_rfc3339(),
    };

    state.add_client(client_info).await;

    let (sender, stream) = sse::channel(200);
    let mut rx = state.event_tx.subscribe();
    let state_clone = state.clone();
    let cid = client_id.clone();

    tokio::spawn(async move {
        // ส่ง welcome event
        let welcome = SseEvent::new(
            "connected",
            serde_json::json!({ "client_id": cid }),
        );
        let _ = sender
            .send(
                sse::Data::new(serde_json::to_string(&welcome).unwrap())
                    .event("connected")
                    .id(&welcome.id),
            )
            .await;

        let mut heartbeat = tokio::time::interval(Duration::from_secs(25));

        loop {
            tokio::select! {
                result = rx.recv() => {
                    match result {
                        Ok(event) => {
                            let json = serde_json::to_string(&event).unwrap_or_default();
                            if sender
                                .send(
                                    sse::Data::new(json)
                                        .event(&event.event_type)
                                        .id(&event.id),
                                )
                                .await
                                .is_err()
                            {
                                break;
                            }
                        }
                        Err(broadcast::error::RecvError::Lagged(n)) => {
                            eprintln!("Client {} lagged by {} events", cid, n);
                        }
                        Err(_) => break,
                    }
                }

                _ = heartbeat.tick() => {
                    let hb = SseEvent::new("heartbeat", serde_json::json!({}));
                    if sender
                        .send(sse::Data::new("{}").event("heartbeat").id(&hb.id))
                        .await
                        .is_err()
                    {
                        break;
                    }
                }
            }
        }

        // Cleanup
        state_clone.remove_client(&cid).await;
        println!("Client {} disconnected", cid);
    });

    Sse::from_channel_stream(stream)
        .with_keep_alive(Duration::from_secs(15))
}

// API endpoint สำหรับส่ง notification
#[derive(Deserialize)]
pub struct SendNotificationRequest {
    pub event_type: String,
    pub payload: serde_json::Value,
}

async fn send_notification(
    state: web::Data<AppState>,
    body: web::Json<SendNotificationRequest>,
) -> HttpResponse {
    let event = SseEvent::new(&body.event_type, body.payload.clone());
    let event_id = event.id.clone();

    state.broadcast(event);

    HttpResponse::Ok().json(serde_json::json!({
        "success": true,
        "event_id": event_id
    }))
}

// Status endpoint
async fn get_status(state: web::Data<AppState>) -> HttpResponse {
    let clients = state.clients.read().await;

    HttpResponse::Ok().json(serde_json::json!({
        "connected_clients": clients.len(),
        "clients": *clients
    }))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let state = web::Data::new(AppState::new());

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .route("/api/notifications", web::get().to(sse_notifications))
            .route("/api/notify", web::post().to(send_notification))
            .route("/api/status", web::get().to(get_status))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### ทดสอบด้วย curl

```bash
# เปิด SSE connection
curl -N http://localhost:8080/api/notifications

# ส่ง notification
curl -X POST http://localhost:8080/api/notify \
  -H "Content-Type: application/json" \
  -d '{"event_type":"user_joined","payload":{"username":"Alice"}}'

# ดู status
curl http://localhost:8080/api/status
```

---

## สรุป

✅ SSE vs WebSocket comparison  
✅ actix-web-lab SSE setup  
✅ Event format (data, id, event, retry)  
✅ Custom event types  
✅ Heartbeat events  
✅ Browser EventSource API  
✅ Reconnection handling  
✅ Live notifications system  

### Exercise

1. เพิ่ม user-specific notifications (ส่งเฉพาะบาง users)
2. ทำ event buffering สำหรับ offline clients
3. เพิ่ม rate limiting ต่อ client
4. Implement event filtering (client เลือก event types ที่ต้องการ)

---

*[← Part 051: WebSockets Real-time](../part_051/README.md) | [Part 053: File Streaming and Upload →](../part_053/README.md)*
