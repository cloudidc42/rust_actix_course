# Part 073: Docker and Containerization

## บทนำ

Docker ช่วยให้เราสามารถ package แอปพลิเคชัน Rust ของเราพร้อมกับ dependencies ทั้งหมดเป็น container image บทนี้จะครอบคลุมการสร้าง Dockerfile ที่ optimized สำหรับ Rust, Docker Compose, และการตั้งค่าสำหรับ production

## 1. Dockerfile สำหรับ Rust (Multi-stage Build)

### 1.1 Basic Multi-stage Dockerfile

```dockerfile
# Dockerfile

# Stage 1: Builder
FROM rust:1.74-slim-bullseye AS builder

# ติดตั้ง dependencies ที่จำเป็น
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# สร้าง working directory
WORKDIR /app

# Copy manifest files ก่อน (สำหรับ layer caching)
COPY Cargo.toml Cargo.lock ./

# สร้าง dummy src เพื่อ cache dependencies
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

# Copy source code จริง
COPY src ./src
COPY migrations ./migrations

# Build แบบ release (touch main.rs เพื่อ force rebuild)
RUN touch src/main.rs
RUN cargo build --release

# Stage 2: Runtime image
FROM debian:bullseye-slim

# ติดตั้ง runtime dependencies เท่านั้น
RUN apt-get update && apt-get install -y \
    libssl1.1 \
    libpq5 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# สร้าง user ที่ไม่ใช่ root
RUN useradd -r -s /bin/false appuser

WORKDIR /app

# Copy binary จาก builder stage
COPY --from=builder /app/target/release/my-api /app/my-api

# Copy migrations และ static files
COPY --from=builder /app/migrations /app/migrations

# เปลี่ยน ownership
RUN chown -R appuser:appuser /app

# Switch ไปใช้ user ที่ไม่ใช่ root
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Run binary
CMD ["/app/my-api"]
```

### 1.2 Multi-stage ด้วย cargo-chef (การ optimize cache)

```dockerfile
# Dockerfile.chef - ใช้ cargo-chef เพื่อ cache dependencies ดียิ่งขึ้น

FROM rust:1.74-slim-bullseye AS chef
RUN cargo install cargo-chef
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Stage: planner - สร้าง recipe
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# Stage: builder - build dependencies ก่อน
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json

# Build dependencies - layer นี้จะถูก cache ถ้า dependencies ไม่เปลี่ยน
RUN cargo chef cook --release --recipe-path recipe.json

# Build application
COPY . .
RUN cargo build --release --bin my-api

# Stage: runtime
FROM debian:bullseye-slim AS runtime

RUN apt-get update && apt-get install -y \
    libssl1.1 \
    libpq5 \
    ca-certificates \
    curl \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -r -s /bin/false appuser
WORKDIR /app

COPY --from=builder /app/target/release/my-api /app/my-api
COPY --from=builder /app/migrations /app/migrations

RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["/app/my-api"]
```

## 2. Optimizing Docker Image Size

### 2.1 ใช้ Alpine Linux

```dockerfile
# Dockerfile.alpine - smaller image

FROM rust:1.74-alpine AS builder

RUN apk add --no-cache \
    musl-dev \
    openssl-dev \
    postgresql-dev \
    pkgconfig

WORKDIR /app

# ตั้งค่า static linking
ENV RUSTFLAGS="-C target-feature=+crt-static"

COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl
RUN rm -rf src

COPY src ./src
RUN touch src/main.rs
RUN cargo build --release --target x86_64-unknown-linux-musl

# ใช้ scratch image (ขนาดเล็กที่สุด!)
FROM scratch AS runtime

COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/my-api /my-api
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080

CMD ["/my-api"]
```

### 2.2 ตรวจสอบขนาด Image

```bash
# Build และดูขนาด
docker build -t my-api:latest .
docker build -t my-api:alpine -f Dockerfile.alpine .

# เปรียบเทียบ
docker images my-api

# วิเคราะห์ layers
docker history my-api:latest

# ใช้ dive เพื่อ analyze
# ติดตั้ง: https://github.com/wagoodman/dive
dive my-api:latest
```

### 2.3 .dockerignore

```dockerignore
# .dockerignore
# Version control
.git
.gitignore
.gitattributes

# Build artifacts
target/
*.o
*.a

# Documentation
docs/
README.md
*.md

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Test and development files
tests/
benches/
examples/
.env
.env.local
.env.development

# CI/CD files
.github/
.gitlab-ci.yml
Jenkinsfile

# Docker files (ไม่จำเป็นใน context)
docker-compose*.yml
Dockerfile*

# Logs
*.log
logs/
```

## 3. docker-compose.yml

### 3.1 Development Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379
      - RUST_LOG=debug
      - APP_ENV=development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./migrations:/app/migrations
    networks:
      - app-network
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes --requirepass redispassword
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redispassword", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - app-network

  # Database admin UI
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres
    networks:
      - app-network
    profiles:
      - debug  # เปิดใช้เฉพาะ debug mode

  # Redis admin UI
  redis-commander:
    image: rediscommander/redis-commander:latest
    environment:
      REDIS_HOSTS: local:redis:6379:0:redispassword
    ports:
      - "8081:8081"
    depends_on:
      - redis
    networks:
      - app-network
    profiles:
      - debug

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

### 3.2 Production Docker Compose

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  api:
    image: ${DOCKER_IMAGE:-my-api}:${VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        max_attempts: 3
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - RUST_LOG=info
      - APP_ENV=production
      - JWT_SECRET=${JWT_SECRET}
    secrets:
      - db_password
      - jwt_secret
    networks:
      - app-network
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.tls=true"
      - "traefik.http.routers.api.tls.certresolver=letsencrypt"

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
      POSTGRES_DB: mydb
    secrets:
      - db_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network
    deploy:
      placement:
        constraints:
          - node.role == manager

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - certbot_data:/etc/letsencrypt:ro
    depends_on:
      - api
    networks:
      - app-network
    restart: always

volumes:
  postgres_data:
  redis_data:
  certbot_data:

networks:
  app-network:
    driver: overlay
    attachable: true

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
```

## 4. Environment Variables ใน Docker

### 4.1 การจัดการ Environment Variables

```bash
# .env (สำหรับ development - อย่า commit!)
DATABASE_URL=postgres://postgres:password@localhost:5432/mydb
REDIS_URL=redis://:redispassword@localhost:6379
JWT_SECRET=super-secret-key-change-in-production
RUST_LOG=debug
PORT=8080
```

```rust
// src/config.rs - Type-safe configuration
use serde::Deserialize;
use std::env;

#[derive(Debug, Deserialize, Clone)]
pub struct Config {
    pub database_url: String,
    pub redis_url: Option<String>,
    pub jwt_secret: String,
    pub rust_log: Option<String>,
    pub port: u16,
    pub app_env: AppEnv,
}

#[derive(Debug, Deserialize, Clone, PartialEq)]
#[serde(rename_all = "lowercase")]
pub enum AppEnv {
    Development,
    Staging,
    Production,
}

impl Config {
    pub fn from_env() -> Result<Self, config::ConfigError> {
        let settings = config::Config::builder()
            // อ่านจาก environment variables
            .add_source(config::Environment::default())
            // Default values
            .set_default("port", 8080)?
            .set_default("app_env", "development")?
            .build()?;
        
        settings.try_deserialize()
    }
    
    pub fn is_production(&self) -> bool {
        self.app_env == AppEnv::Production
    }
}
```

### 4.2 Secrets Management

```yaml
# docker-compose.secrets.yml - ใช้ Docker secrets
version: '3.8'

services:
  api:
    image: my-api:latest
    environment:
      DATABASE_URL: "postgres://postgres@postgres/mydb"
    secrets:
      - db_password
      - jwt_secret
    # แอปอ่าน secrets จาก /run/secrets/

secrets:
  db_password:
    file: ./secrets/db_password.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
```

```rust
// src/secrets.rs - อ่าน Docker secrets
use std::fs;
use std::path::Path;

pub fn read_secret(name: &str) -> Option<String> {
    let path = format!("/run/secrets/{}", name);
    if Path::new(&path).exists() {
        fs::read_to_string(&path)
            .ok()
            .map(|s| s.trim().to_string())
    } else {
        std::env::var(name).ok()
    }
}

// ใช้งาน
fn get_db_password() -> String {
    read_secret("db_password")
        .or_else(|| std::env::var("DB_PASSWORD").ok())
        .expect("Database password required")
}
```

## 5. Docker Networks

### 5.1 Network Configuration

```yaml
# docker-compose.networks.yml
version: '3.8'

services:
  api:
    networks:
      - frontend      # รับ traffic จากภายนอก
      - backend       # คุยกับ DB/Cache
    
  postgres:
    networks:
      - backend       # เข้าถึงได้แค่จาก backend network
    
  redis:
    networks:
      - backend

  nginx:
    networks:
      - frontend
    ports:
      - "80:80"

networks:
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  backend:
    driver: bridge
    internal: true  # ไม่มี internet access
    ipam:
      config:
        - subnet: 172.20.1.0/24
```

```bash
# สร้าง external network
docker network create --driver bridge app-network

# ดู networks
docker network ls

# Inspect network
docker network inspect app-network

# Connect container กับ network
docker network connect app-network my-container
```

## 6. Volume Mounts สำหรับ Development

### 6.1 Hot Reload Development Setup

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  api-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev  # Development Dockerfile
    ports:
      - "8080:8080"
    volumes:
      - .:/app                    # Mount source code
      - cargo_cache:/usr/local/cargo/registry  # Cache cargo packages
      - target_cache:/app/target  # Cache build artifacts
    environment:
      - DATABASE_URL=postgres://postgres:password@postgres:5432/mydb
      - RUST_LOG=debug
      - CARGO_HOME=/usr/local/cargo
    command: cargo watch -x run   # hot reload
    depends_on:
      - postgres
    networks:
      - dev-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
    networks:
      - dev-network

volumes:
  cargo_cache:
  target_cache:
  postgres_dev_data:

networks:
  dev-network:
    driver: bridge
```

```dockerfile
# Dockerfile.dev
FROM rust:1.74-slim-bullseye

RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้ง cargo-watch สำหรับ hot reload
RUN cargo install cargo-watch

WORKDIR /app

# ไม่ต้อง copy source - จะ mount จาก host
EXPOSE 8080

CMD ["cargo", "watch", "-x", "run"]
```

## 7. .dockerignore ที่สมบูรณ์

```dockerignore
# .dockerignore - complete version

# VCS
.git
.gitignore
.gitattributes
.gitmodules

# Build outputs
target/
**/*.rs.bk
*.pdb

# Test artifacts
*.profraw
coverage/
tarpaulin-report.html

# IDE and editor files
.vscode/
.idea/
*.swp
*.swo
*~
.editorconfig

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V500
.Trashes
ehthumbs.db
Thumbs.db

# Environment files
.env
.env.*
!.env.example
*.local

# Documentation
docs/
*.md
!README.md  # อาจต้องการ README ใน image

# Test files
tests/
benches/
examples/
**/*_test.rs

# CI/CD
.github/
.gitlab-ci.yml
Jenkinsfile
.travis.yml

# Docker files (ไม่จำเป็นใน build context)
docker-compose*.yml
Dockerfile*

# Log files
*.log
logs/

# Temporary files
tmp/
temp/
*.tmp

# Package manager files
node_modules/  # ถ้ามี frontend
yarn.lock
package-lock.json

# Secrets
secrets/
*.pem
*.key
*.cert
*.crt
!certificates/  # ถ้าต้องการ certs บางอย่าง
```

## 8. Practical: Production-ready Docker Setup

### 8.1 Complete Application Structure

```
my-app/
├── src/
│   ├── main.rs
│   ├── config.rs
│   ├── handlers/
│   └── models/
├── migrations/
├── Cargo.toml
├── Cargo.lock
├── Dockerfile               # Production
├── Dockerfile.dev           # Development
├── .dockerignore
├── docker-compose.yml       # Development
├── docker-compose.prod.yml  # Production
├── nginx.conf               # Nginx config
├── .env.example
└── scripts/
    ├── build.sh
    ├── deploy.sh
    └── healthcheck.sh
```

### 8.2 nginx.conf

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api_backend {
        least_conn;  # Load balancing
        server api:8080;
        # ถ้ามีหลาย instances:
        # server api_1:8080;
        # server api_2:8080;
        # server api_3:8080;
    }

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

    server {
        listen 80;
        server_name api.example.com;
        
        # Redirect HTTP to HTTPS
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        ssl_certificate /etc/letsencrypt/live/api.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/api.example.com/privkey.pem;
        
        # Security headers
        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-XSS-Protection "1; mode=block";
        add_header X-Content-Type-Options "nosniff";
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        location / {
            limit_req zone=api_limit burst=20 nodelay;
            
            proxy_pass http://api_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;
            proxy_send_timeout 60s;
        }

        # Health check (bypass rate limiting)
        location /health {
            proxy_pass http://api_backend;
            access_log off;
        }
    }
}
```

### 8.3 Health Check Endpoint

```rust
// src/handlers/health.rs
use actix_web::{web, HttpResponse};
use sqlx::PgPool;
use serde::Serialize;
use std::time::{Duration, Instant};

#[derive(Serialize)]
struct HealthStatus {
    status: String,
    database: DatabaseHealth,
    uptime_seconds: u64,
    version: String,
}

#[derive(Serialize)]
struct DatabaseHealth {
    connected: bool,
    response_time_ms: u64,
}

static START_TIME: std::sync::OnceLock<Instant> = std::sync::OnceLock::new();

pub async fn health_check(db: web::Data<PgPool>) -> HttpResponse {
    let start = START_TIME.get_or_init(Instant::now);
    let uptime = start.elapsed().as_secs();
    
    // Check database
    let db_start = Instant::now();
    let db_connected = sqlx::query!("SELECT 1 as x")
        .fetch_one(db.get_ref())
        .await
        .is_ok();
    let db_response_time = db_start.elapsed().as_millis() as u64;
    
    let status = if db_connected { "healthy" } else { "degraded" };
    
    let health = HealthStatus {
        status: status.to_string(),
        database: DatabaseHealth {
            connected: db_connected,
            response_time_ms: db_response_time,
        },
        uptime_seconds: uptime,
        version: env!("CARGO_PKG_VERSION").to_string(),
    };
    
    if db_connected {
        HttpResponse::Ok().json(health)
    } else {
        HttpResponse::ServiceUnavailable().json(health)
    }
}

// ใน main.rs
// .route("/health", web::get().to(health_check))
```

### 8.4 Build Script

```bash
#!/bin/bash
# scripts/build.sh

set -e

VERSION=${1:-latest}
REGISTRY=${DOCKER_REGISTRY:-ghcr.io/myorg}
IMAGE_NAME="my-api"

echo "Building $IMAGE_NAME:$VERSION"

# Build image
docker build \
    --tag "$REGISTRY/$IMAGE_NAME:$VERSION" \
    --tag "$REGISTRY/$IMAGE_NAME:latest" \
    --build-arg VERSION="$VERSION" \
    --build-arg BUILD_DATE="$(date -u +'%Y-%m-%dT%H:%M:%SZ')" \
    --file Dockerfile \
    .

echo "Build complete: $REGISTRY/$IMAGE_NAME:$VERSION"

# Run basic tests
echo "Running smoke test..."
docker run --rm \
    -e DATABASE_URL=postgres://test:test@localhost/test \
    -e JWT_SECRET=test-secret \
    "$REGISTRY/$IMAGE_NAME:$VERSION" \
    /app/my-api --version

echo "Smoke test passed!"
```

### 8.5 Deploy Script

```bash
#!/bin/bash
# scripts/deploy.sh

set -e

COMPOSE_FILE=${COMPOSE_FILE:-docker-compose.prod.yml}
VERSION=${1:-latest}

echo "Deploying version: $VERSION"

# Export version for docker-compose
export VERSION=$VERSION

# Pull latest images
docker-compose -f "$COMPOSE_FILE" pull

# Deploy with zero-downtime (ถ้าใช้ swarm)
if docker info | grep -q "Swarm: active"; then
    docker stack deploy \
        --compose-file "$COMPOSE_FILE" \
        --with-registry-auth \
        myapp
else
    # Standard deployment
    docker-compose -f "$COMPOSE_FILE" up -d --no-deps api
fi

echo "Waiting for health check..."
sleep 10

# Check health
if curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "Deployment successful!"
else
    echo "Health check failed! Rolling back..."
    docker-compose -f "$COMPOSE_FILE" rollback api
    exit 1
fi
```

### 8.6 Makefile สำหรับสะดวกใช้

```makefile
# Makefile

.PHONY: build dev test prod clean

# Development
dev:
	docker-compose up --build

dev-detach:
	docker-compose up -d --build

# Production
prod:
	docker-compose -f docker-compose.prod.yml up -d

# Build
build:
	docker build -t my-api:latest .

build-alpine:
	docker build -f Dockerfile.alpine -t my-api:alpine .

# Test
test:
	docker-compose run --rm api cargo test

# Logs
logs:
	docker-compose logs -f api

# Clean
clean:
	docker-compose down -v
	docker system prune -f

# Shell
shell:
	docker-compose exec api sh

# DB
db-migrate:
	docker-compose exec api sqlx migrate run

db-reset:
	docker-compose exec postgres psql -U postgres -c "DROP DATABASE IF EXISTS mydb;"
	docker-compose exec postgres psql -U postgres -c "CREATE DATABASE mydb;"
	$(MAKE) db-migrate
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Multi-stage Dockerfile** - ลด image size ด้วย builder + runtime stages
2. **cargo-chef** - optimize cache สำหรับ Rust dependencies
3. **docker-compose** - orchestrate หลาย services
4. **Environment variables** - จัดการ configuration อย่างปลอดภัย
5. **Docker networks** - แยก frontend/backend networks
6. **Volume mounts** - hot reload สำหรับ development
7. **Production setup** - nginx, health checks, deploy scripts

---

[⬅️ Part 072: Load Testing](../part_072/README.md) | [➡️ Part 074: CI/CD with GitHub Actions](../part_074/README.md)
