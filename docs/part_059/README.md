# Part 059: Message Queue with RabbitMQ 🐰

## 🎯 เป้าหมายของ Part นี้

- lapin crate (AMQP client)
- Connection และ channel
- Exchange types (direct, fanout, topic)
- Publishing messages
- Consuming messages
- Acknowledgment (ack/nack)
- Dead letter exchange
- สร้าง Event-driven architecture

---

## 1. Setup

```toml
# Cargo.toml
[dependencies]
actix-web = "4"
lapin = "2"
tokio = { version = "1", features = ["full"] }
tokio-amqp = "2"
futures-lite = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
uuid = { version = "1", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
thiserror = "1"
anyhow = "1"
tracing = "0.1"
tracing-subscriber = "0.3"
```

---

## 2. Connection Setup

```rust
// src/rabbitmq/connection.rs
use lapin::{
    Connection, ConnectionProperties, Channel,
    options::*,
    types::FieldTable,
};

pub async fn create_connection(
    uri: &str,
) -> anyhow::Result<Connection> {
    let conn = Connection::connect(
        uri,
        ConnectionProperties::default()
            .with_executor(tokio_executor_trait::Tokio::current())
            .with_reactor(tokio_reactor_trait::Tokio),
    )
    .await?;

    println!("[RabbitMQ] Connected to: {}", uri);

    Ok(conn)
}

pub async fn create_channel(conn: &Connection) -> anyhow::Result<Channel> {
    let channel = conn.create_channel().await?;
    println!("[RabbitMQ] Channel created");
    Ok(channel)
}

// Connection pool (ใช้ Arc<RwLock> สำหรับ shared connection)
use std::sync::Arc;
use tokio::sync::RwLock;

pub struct RabbitMQPool {
    connection: Arc<RwLock<Option<Connection>>>,
    uri: String,
}

impl RabbitMQPool {
    pub fn new(uri: &str) -> Self {
        Self {
            connection: Arc::new(RwLock::new(None)),
            uri: uri.to_string(),
        }
    }

    pub async fn get_channel(&self) -> anyhow::Result<Channel> {
        let conn = self.connection.read().await;

        if let Some(ref c) = *conn {
            if c.status().connected() {
                return Ok(c.create_channel().await?);
            }
        }

        drop(conn);

        // Reconnect
        let new_conn = create_connection(&self.uri).await?;
        let channel = new_conn.create_channel().await?;
        *self.connection.write().await = Some(new_conn);

        Ok(channel)
    }
}
```

---

## 3. Exchange Types

```rust
// src/rabbitmq/exchanges.rs
use lapin::{
    Channel,
    options::ExchangeDeclareOptions,
    types::FieldTable,
    ExchangeKind,
};

// Exchange types
pub enum ExchangeType {
    Direct,   // Route ตาม routing key ตรงๆ
    Fanout,   // Broadcast ไปทุก queues
    Topic,    // Pattern matching (*, #)
    Headers,  // Match จาก message headers
}

// Setup exchanges
pub async fn setup_exchanges(channel: &Channel) -> anyhow::Result<()> {
    // 1. Direct exchange - route ตาม key ตรงๆ
    channel
        .exchange_declare(
            "direct.notifications",
            ExchangeKind::Direct,
            ExchangeDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    // 2. Fanout exchange - broadcast
    channel
        .exchange_declare(
            "fanout.events",
            ExchangeKind::Fanout,
            ExchangeDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    // 3. Topic exchange - pattern matching
    channel
        .exchange_declare(
            "topic.app",
            ExchangeKind::Topic,
            ExchangeDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    // 4. Dead Letter Exchange
    channel
        .exchange_declare(
            "dead.letter",
            ExchangeKind::Direct,
            ExchangeDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    println!("[RabbitMQ] Exchanges declared");
    Ok(())
}

// Setup queues with bindings
pub async fn setup_queues(channel: &Channel) -> anyhow::Result<()> {
    use lapin::options::QueueDeclareOptions;

    // Dead letter queue ก่อน
    channel
        .queue_declare(
            "q.dead.letter",
            QueueDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    // Bind dead letter queue
    channel
        .queue_bind(
            "q.dead.letter",
            "dead.letter",
            "dead",
            Default::default(),
            FieldTable::default(),
        )
        .await?;

    // Email notification queue (direct)
    let mut args = FieldTable::default();
    args.insert(
        "x-dead-letter-exchange".into(),
        lapin::types::AMQPValue::LongString("dead.letter".into()),
    );
    args.insert(
        "x-dead-letter-routing-key".into(),
        lapin::types::AMQPValue::LongString("dead".into()),
    );
    args.insert(
        "x-message-ttl".into(),
        lapin::types::AMQPValue::LongInt(86400000), // 24h
    );

    channel
        .queue_declare(
            "q.email.notifications",
            QueueDeclareOptions {
                durable: true,
                ..Default::default()
            },
            args,
        )
        .await?;

    channel
        .queue_bind(
            "q.email.notifications",
            "direct.notifications",
            "email",
            Default::default(),
            FieldTable::default(),
        )
        .await?;

    // User events queue (fanout)
    channel
        .queue_declare(
            "q.user.events",
            QueueDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    channel
        .queue_bind(
            "q.user.events",
            "fanout.events",
            "", // fanout ไม่ใช้ routing key
            Default::default(),
            FieldTable::default(),
        )
        .await?;

    // Order events (topic)
    channel
        .queue_declare(
            "q.order.created",
            QueueDeclareOptions {
                durable: true,
                ..Default::default()
            },
            FieldTable::default(),
        )
        .await?;

    // Bind ด้วย topic pattern: order.*
    channel
        .queue_bind(
            "q.order.created",
            "topic.app",
            "order.*",
            Default::default(),
            FieldTable::default(),
        )
        .await?;

    println!("[RabbitMQ] Queues declared and bound");
    Ok(())
}
```

---

## 4. Publishing Messages

```rust
// src/rabbitmq/publisher.rs
use lapin::{
    BasicProperties, Channel,
    options::BasicPublishOptions,
};
use serde::Serialize;

pub struct Publisher {
    channel: Channel,
}

impl Publisher {
    pub fn new(channel: Channel) -> Self {
        Self { channel }
    }

    // Publish JSON message
    pub async fn publish_json<T: Serialize>(
        &self,
        exchange: &str,
        routing_key: &str,
        payload: &T,
    ) -> anyhow::Result<()> {
        let json = serde_json::to_vec(payload)?;

        self.channel
            .basic_publish(
                exchange,
                routing_key,
                BasicPublishOptions::default(),
                &json,
                BasicProperties::default()
                    .with_content_type("application/json".into())
                    .with_delivery_mode(2), // persistent
            )
            .await?
            .await?; // wait for confirm

        Ok(())
    }

    // Publish with headers
    pub async fn publish_with_headers<T: Serialize>(
        &self,
        exchange: &str,
        routing_key: &str,
        payload: &T,
        headers: Vec<(&str, &str)>,
    ) -> anyhow::Result<()> {
        let json = serde_json::to_vec(payload)?;

        let mut field_table = lapin::types::FieldTable::default();
        for (key, value) in headers {
            field_table.insert(
                key.into(),
                lapin::types::AMQPValue::LongString(value.into()),
            );
        }

        self.channel
            .basic_publish(
                exchange,
                routing_key,
                BasicPublishOptions::default(),
                &json,
                BasicProperties::default()
                    .with_content_type("application/json".into())
                    .with_delivery_mode(2)
                    .with_headers(field_table),
            )
            .await?
            .await?;

        Ok(())
    }

    // Publish email event
    pub async fn send_email_notification(
        &self,
        to: &str,
        subject: &str,
        body: &str,
    ) -> anyhow::Result<()> {
        let payload = serde_json::json!({
            "to": to,
            "subject": subject,
            "body": body,
            "timestamp": chrono::Utc::now().to_rfc3339()
        });

        self.publish_json("direct.notifications", "email", &payload).await
    }

    // Broadcast user event (fanout)
    pub async fn broadcast_user_event(
        &self,
        event_type: &str,
        user_id: &str,
        data: serde_json::Value,
    ) -> anyhow::Result<()> {
        let payload = serde_json::json!({
            "event_type": event_type,
            "user_id": user_id,
            "data": data,
            "timestamp": chrono::Utc::now().to_rfc3339()
        });

        self.publish_json("fanout.events", "", &payload).await
    }

    // Topic publish
    pub async fn publish_order_event(
        &self,
        event_type: &str, // "created", "updated", "cancelled"
        order_id: &str,
        data: serde_json::Value,
    ) -> anyhow::Result<()> {
        let routing_key = format!("order.{}", event_type);
        let payload = serde_json::json!({
            "event_type": event_type,
            "order_id": order_id,
            "data": data,
            "timestamp": chrono::Utc::now().to_rfc3339()
        });

        self.publish_json("topic.app", &routing_key, &payload).await
    }
}
```

---

## 5. Consuming Messages

```rust
// src/rabbitmq/consumer.rs
use lapin::{
    Channel,
    options::{BasicAckOptions, BasicNackOptions, BasicConsumeOptions},
    types::FieldTable,
    message::Delivery,
};
use futures_lite::stream::StreamExt;

pub struct Consumer {
    channel: Channel,
}

impl Consumer {
    pub fn new(channel: Channel) -> Self {
        Self { channel }
    }

    // Basic consumer
    pub async fn consume_emails(&self) -> anyhow::Result<()> {
        let mut consumer = self
            .channel
            .basic_consume(
                "q.email.notifications",
                "email_consumer",
                BasicConsumeOptions::default(),
                FieldTable::default(),
            )
            .await?;

        println!("[Consumer] Waiting for email messages...");

        while let Some(delivery) = consumer.next().await {
            match delivery {
                Ok(delivery) => {
                    match self.process_email(&delivery).await {
                        Ok(_) => {
                            // ACK - message processed successfully
                            delivery
                                .ack(BasicAckOptions::default())
                                .await?;
                        }
                        Err(e) => {
                            eprintln!("[Consumer] Processing failed: {}", e);

                            // NACK - requeue=false หรือ true
                            // false = ส่งไป dead letter queue
                            // true = requeue กลับไป
                            delivery
                                .nack(BasicNackOptions {
                                    requeue: false,
                                    ..Default::default()
                                })
                                .await?;
                        }
                    }
                }
                Err(e) => {
                    eprintln!("[Consumer] Delivery error: {}", e);
                }
            }
        }

        Ok(())
    }

    async fn process_email(&self, delivery: &Delivery) -> anyhow::Result<()> {
        let payload: serde_json::Value = serde_json::from_slice(&delivery.data)?;

        let to = payload["to"].as_str().unwrap_or("unknown");
        let subject = payload["subject"].as_str().unwrap_or("No subject");

        println!("[Email] Sending to: {} - Subject: {}", to, subject);

        // TODO: ส่ง email จริงๆ ด้วย lettre
        // email_service.send(...).await?;

        Ok(())
    }

    // Concurrent consumer พร้อม prefetch
    pub async fn consume_with_prefetch(
        channel: Channel,
        queue_name: &str,
        consumer_tag: &str,
        prefetch_count: u16,
        handler: impl Fn(Delivery) -> std::pin::Pin<Box<dyn std::future::Future<Output = anyhow::Result<()>> + Send>> + Send + Sync + 'static,
    ) -> anyhow::Result<()> {
        use lapin::options::BasicQosOptions;
        use std::sync::Arc;

        // Set QoS (prefetch)
        channel
            .basic_qos(prefetch_count, BasicQosOptions::default())
            .await?;

        let mut consumer = channel
            .basic_consume(
                queue_name,
                consumer_tag,
                BasicConsumeOptions::default(),
                FieldTable::default(),
            )
            .await?;

        let handler = Arc::new(handler);

        while let Some(delivery) = consumer.next().await {
            match delivery {
                Ok(delivery) => {
                    let h = handler.clone();
                    tokio::spawn(async move {
                        match h(delivery.clone()).await {
                            Ok(_) => {
                                let _ = delivery.ack(BasicAckOptions::default()).await;
                            }
                            Err(e) => {
                                eprintln!("[Consumer] Error: {}", e);
                                let _ = delivery
                                    .nack(BasicNackOptions {
                                        requeue: false,
                                        ..Default::default()
                                    })
                                    .await;
                            }
                        }
                    });
                }
                Err(e) => eprintln!("[Consumer] Error: {}", e),
            }
        }

        Ok(())
    }
}
```

---

## 6. Practical: Event-Driven Architecture

```rust
// src/main.rs - Complete event-driven system
use actix_web::{web, App, HttpServer, HttpResponse};
use lapin::Connection;
use std::sync::Arc;
use tokio::sync::RwLock;

mod rabbitmq {
    pub mod connection;
    pub mod exchanges;
    pub mod publisher;
    pub mod consumer;
}

pub struct AppState {
    pub publisher: Arc<rabbitmq::publisher::Publisher>,
}

// API: Create order → publish event
async fn create_order(
    state: web::Data<AppState>,
    body: web::Json<CreateOrderRequest>,
) -> HttpResponse {
    let order_id = uuid::Uuid::new_v4().to_string();

    // Save order to DB (simplified)
    println!("[API] Creating order: {}", order_id);

    // Publish event
    let data = serde_json::json!({
        "order_id": order_id,
        "items": body.items,
        "total": body.total,
        "customer_id": body.customer_id,
    });

    if let Err(e) = state
        .publisher
        .publish_order_event("created", &order_id, data)
        .await
    {
        eprintln!("[API] Failed to publish event: {}", e);
    }

    // Also send email notification
    if let Err(e) = state
        .publisher
        .send_email_notification(
            &body.customer_email,
            "Order Confirmed",
            &format!("Your order {} has been created", order_id),
        )
        .await
    {
        eprintln!("[API] Failed to send email notification: {}", e);
    }

    HttpResponse::Created().json(serde_json::json!({
        "order_id": order_id,
        "status": "created"
    }))
}

#[derive(serde::Deserialize)]
struct CreateOrderRequest {
    customer_id: String,
    customer_email: String,
    items: Vec<serde_json::Value>,
    total: f64,
}

// Worker: consume order events
async fn start_order_worker(channel: lapin::Channel) {
    use lapin::{message::Delivery, options::*};
    use futures_lite::StreamExt;

    channel
        .basic_qos(10, BasicQosOptions::default())
        .await
        .expect("Failed to set QoS");

    let mut consumer = channel
        .basic_consume(
            "q.order.created",
            "order_worker",
            BasicConsumeOptions::default(),
            lapin::types::FieldTable::default(),
        )
        .await
        .expect("Failed to create consumer");

    println!("[OrderWorker] Started, waiting for orders...");

    while let Some(delivery) = consumer.next().await {
        if let Ok(delivery) = delivery {
            let payload: serde_json::Value =
                serde_json::from_slice(&delivery.data).unwrap_or_default();

            println!(
                "[OrderWorker] Processing order: {}",
                payload["order_id"]
            );

            // Process order...
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;

            delivery
                .ack(BasicAckOptions::default())
                .await
                .expect("Failed to ack");
        }
    }
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    tracing_subscriber::init();

    let amqp_uri = std::env::var("AMQP_URL")
        .unwrap_or("amqp://guest:guest@localhost:5672/%2f".to_string());

    let conn = rabbitmq::connection::create_connection(&amqp_uri)
        .await
        .expect("Failed to connect to RabbitMQ");

    let channel = conn.create_channel().await.expect("Channel error");
    rabbitmq::exchanges::setup_exchanges(&channel)
        .await
        .expect("Exchange setup error");
    rabbitmq::exchanges::setup_queues(&channel)
        .await
        .expect("Queue setup error");

    // Publisher channel
    let pub_channel = conn.create_channel().await.expect("Publisher channel error");
    let publisher = Arc::new(rabbitmq::publisher::Publisher::new(pub_channel));

    // Start consumers in background
    let worker_channel = conn.create_channel().await.expect("Worker channel error");
    tokio::spawn(async move {
        start_order_worker(worker_channel).await;
    });

    let email_channel = conn.create_channel().await.expect("Email channel error");
    let email_consumer = rabbitmq::consumer::Consumer::new(email_channel);
    tokio::spawn(async move {
        email_consumer.consume_emails().await.ok();
    });

    // Start HTTP API
    let state = web::Data::new(AppState { publisher });

    println!("[API] Starting on 0.0.0.0:8080");

    HttpServer::new(move || {
        App::new()
            .app_data(state.clone())
            .route("/api/orders", web::post().to(create_order))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

---

## สรุป

✅ lapin AMQP client  
✅ Connection และ channel management  
✅ Exchange types (direct, fanout, topic)  
✅ Publishing messages  
✅ Consuming messages พร้อม ack/nack  
✅ Dead letter exchange  
✅ Event-driven architecture  

### Exercise

1. เพิ่ม message retry ด้วย x-retry-count header
2. สร้าง monitoring dashboard สำหรับ queue stats
3. Implement saga pattern ด้วย message choreography
4. เพิ่ม message schema validation

---

*[← Part 058: gRPC with Tonic](../part_058/README.md) | [Part 060: Webhooks →](../part_060/README.md)*
