# Part 076: Kubernetes Basics for Rust APIs

## บทนำ

Kubernetes (K8s) เป็น container orchestration platform ที่ช่วยจัดการ deployment, scaling, และ availability ของแอปพลิเคชัน บทนี้จะครอบคลุมการสร้าง Kubernetes manifests สำหรับ Rust API

## 1. Deployment Manifest

### 1.1 Basic Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rust-api
  namespace: production
  labels:
    app: rust-api
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rust-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # สร้าง pod ใหม่ได้สูงสุด 1 ตัวระหว่าง update
      maxUnavailable: 0    # ไม่มี pod หยุดทำงาน (zero-downtime)
  template:
    metadata:
      labels:
        app: rust-api
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
      
      # Security context สำหรับ pod
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      containers:
        - name: rust-api
          image: ghcr.io/myorg/rust-api:1.0.0
          imagePullPolicy: IfNotPresent
          
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          
          # Environment variables จาก ConfigMap และ Secret
          env:
            - name: APP_ENV
              value: "production"
            - name: RUST_LOG
              value: "info"
            - name: PORT
              value: "8080"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: rust-api-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: rust-api-secrets
                  key: jwt-secret
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: rust-api-config
                  key: redis-url
          
          # Resource limits
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          
          # Readiness probe
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Liveness probe
          livenessProbe:
            httpGet:
              path: /live
              port: http
            initialDelaySeconds: 30
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          
          # Startup probe (รอ app เริ่มต้น)
          startupProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 12  # รอสูงสุด 60 วินาที
          
          # Security context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          # Mount points สำหรับ writable directories
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-logs
              mountPath: /app/logs
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-logs
          emptyDir: {}
      
      # Pull secrets (ถ้า image อยู่ใน private registry)
      imagePullSecrets:
        - name: ghcr-credentials
```

## 2. Service Manifest

### 2.1 ClusterIP Service (ภายใน cluster)

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: rust-api
  namespace: production
  labels:
    app: rust-api
spec:
  type: ClusterIP
  selector:
    app: rust-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
```

### 2.2 NodePort Service (สำหรับ testing)

```yaml
# k8s/service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: rust-api-nodeport
  namespace: staging
spec:
  type: NodePort
  selector:
    app: rust-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # Access ด้วย <node-ip>:30080
```

### 2.3 LoadBalancer Service (Cloud)

```yaml
# k8s/service-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: rust-api-lb
  namespace: production
  annotations:
    # AWS EKS
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    # GKE
    cloud.google.com/load-balancer-type: External
spec:
  type: LoadBalancer
  selector:
    app: rust-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
```

## 3. ConfigMap และ Secrets

### 3.1 ConfigMap

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rust-api-config
  namespace: production
data:
  # Key-value config
  redis-url: "redis://redis-service:6379"
  log-level: "info"
  max-connections: "20"
  
  # Config file
  app-config.toml: |
    [database]
    max_connections = 20
    min_connections = 5
    
    [cache]
    ttl_seconds = 300
    max_entries = 10000
    
    [server]
    workers = 4
    keep_alive = 75
```

### 3.2 Secret

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: rust-api-secrets
  namespace: production
type: Opaque
# ค่าต้องเป็น base64 encoded
# echo -n "value" | base64
data:
  database-url: cG9zdGdyZXM6Ly91c2VyOnBhc3N3b3JkQHBvc3RncmVzOjU0MzIvbXlkYg==
  jwt-secret: c3VwZXItc2VjcmV0LWtleS1mb3ItcHJvZHVjdGlvbg==
  redis-password: cmVkaXNwYXNzd29yZA==
```

```bash
# สร้าง Secret จาก command line (ดีกว่า เพราะไม่ต้อง encode เอง)
kubectl create secret generic rust-api-secrets \
  --namespace production \
  --from-literal=database-url="postgres://user:password@postgres:5432/mydb" \
  --from-literal=jwt-secret="super-secret-key-for-production" \
  --from-literal=redis-password="redispassword"

# สร้าง Secret จาก file
kubectl create secret generic tls-certs \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key

# ดู Secrets
kubectl get secrets -n production
kubectl describe secret rust-api-secrets -n production

# Decode secret value
kubectl get secret rust-api-secrets -n production \
  -o jsonpath='{.data.database-url}' | base64 -d
```

### 3.3 External Secrets (ดีกว่าสำหรับ production)

```yaml
# k8s/external-secret.yaml
# ใช้ External Secrets Operator เพื่อ sync กับ AWS Secrets Manager / HashiCorp Vault
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rust-api-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: rust-api-secrets
    creationPolicy: Owner
  data:
    - secretKey: database-url
      remoteRef:
        key: production/rust-api
        property: DATABASE_URL
    - secretKey: jwt-secret
      remoteRef:
        key: production/rust-api
        property: JWT_SECRET
```

## 4. Resource Limits

### 4.1 LimitRange (กำหนด defaults สำหรับ namespace)

```yaml
# k8s/limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:          # ค่า default ถ้าไม่ได้กำหนด
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:   # ค่า request default
        cpu: "100m"
        memory: "64Mi"
      max:              # ค่าสูงสุด
        cpu: "2"
        memory: "2Gi"
      min:              # ค่าต่ำสุด
        cpu: "50m"
        memory: "32Mi"
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"
```

### 4.2 ResourceQuota (จำกัดทรัพยากรทั้ง namespace)

```yaml
# k8s/resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Pod limits
    pods: "20"
    
    # Compute resources
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "8"
    limits.memory: "8Gi"
    
    # Storage
    requests.storage: "50Gi"
    persistentvolumeclaims: "10"
    
    # Service limits
    services.loadbalancers: "2"
    services.nodeports: "0"
```

## 5. Horizontal Pod Autoscaler (HPA)

### 5.1 CPU-based HPA

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rust-api-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rust-api
  
  minReplicas: 2
  maxReplicas: 10
  
  metrics:
    # Scale based on CPU usage
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # Scale up เมื่อ CPU > 70%
    
    # Scale based on Memory usage
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # รอ 60 วินาทีก่อน scale up
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60  # เพิ่มได้สูงสุด 2 pods ต่อ 60 วินาที
    scaleDown:
      stabilizationWindowSeconds: 300   # รอ 5 นาทีก่อน scale down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120  # ลดได้สูงสุด 1 pod ต่อ 2 นาที
```

### 5.2 Custom Metrics HPA

```yaml
# k8s/hpa-custom.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rust-api-hpa-custom
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rust-api
  
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
    # Custom metric จาก Prometheus
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # Scale เมื่อ > 100 req/s per pod
    
    # External metric (queue length)
    - type: External
      external:
        metric:
          name: queue_length
          selector:
            matchLabels:
              queue: "api-requests"
        target:
          type: AverageValue
          averageValue: "100"
```

## 6. Ingress

### 6.1 NGINX Ingress

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rust-api-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    
    # TLS redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-connections: "100"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-rpm: "1000"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    
    # Cert-manager
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  tls:
    - hosts:
        - api.myapp.com
      secretName: api-tls-cert
  
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: rust-api
                port:
                  number: 80
          
          - path: /health
            pathType: Exact
            backend:
              service:
                name: rust-api
                port:
                  number: 80
```

### 6.2 Traefik Ingress

```yaml
# k8s/ingress-traefik.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rust-api-traefik
  namespace: production
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
    
    # Middleware
    traefik.ingress.kubernetes.io/router.middlewares: >
      production-rate-limit@kubernetescrd,
      production-compress@kubernetescrd
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: rust-api
                port:
                  number: 80
```

## 7. Readiness/Liveness Probes

### 7.1 Rust Implementation

```rust
// src/handlers/probes.rs
use actix_web::{web, HttpResponse};
use sqlx::PgPool;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

// Global state สำหรับควบคุม readiness
pub struct AppReadiness {
    is_ready: AtomicBool,
}

impl AppReadiness {
    pub fn new() -> Arc<Self> {
        Arc::new(AppReadiness {
            is_ready: AtomicBool::new(false),
        })
    }
    
    pub fn set_ready(&self) {
        self.is_ready.store(true, Ordering::SeqCst);
        log::info!("Application is now ready to serve traffic");
    }
    
    pub fn set_not_ready(&self) {
        self.is_ready.store(false, Ordering::SeqCst);
        log::warn!("Application is no longer ready");
    }
    
    pub fn is_ready(&self) -> bool {
        self.is_ready.load(Ordering::SeqCst)
    }
}

// Readiness probe - K8s จะ route traffic มาเมื่อ ready
pub async fn readiness(
    db: web::Data<PgPool>,
    readiness: web::Data<Arc<AppReadiness>>,
) -> HttpResponse {
    if !readiness.is_ready() {
        return HttpResponse::ServiceUnavailable()
            .body("Application is not ready");
    }
    
    // Check database connection
    match sqlx::query!("SELECT 1 AS check")
        .fetch_one(db.get_ref())
        .await
    {
        Ok(_) => HttpResponse::Ok().body("ready"),
        Err(e) => {
            log::error!("Database not accessible: {}", e);
            HttpResponse::ServiceUnavailable()
                .body("Database not accessible")
        }
    }
}

// Liveness probe - K8s จะ restart pod ถ้า fail
pub async fn liveness() -> HttpResponse {
    HttpResponse::Ok().body("alive")
}

// Startup probe - รอ app เริ่มต้นสำเร็จ
pub async fn startup(
    db: web::Data<PgPool>,
) -> HttpResponse {
    match sqlx::query!("SELECT 1 AS check")
        .fetch_one(db.get_ref())
        .await
    {
        Ok(_) => HttpResponse::Ok().body("started"),
        Err(_) => HttpResponse::ServiceUnavailable().body("not started"),
    }
}
```

```rust
// src/main.rs - ใช้งาน probes
use std::sync::Arc;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let readiness = handlers::probes::AppReadiness::new();
    let readiness_data = web::Data::new(Arc::clone(&readiness));
    
    let server = HttpServer::new(move || {
        App::new()
            .app_data(readiness_data.clone())
            .route("/ready", web::get().to(handlers::probes::readiness))
            .route("/live", web::get().to(handlers::probes::liveness))
            .route("/startup", web::get().to(handlers::probes::startup))
    })
    .bind("0.0.0.0:8080")?
    .run();
    
    // Mark as ready หลังจาก setup เสร็จ
    readiness.set_ready();
    
    server.await
}
```

## 8. Practical: K8s Manifests สำหรับ API

### 8.1 Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    name: production
    environment: production
```

### 8.2 PostgreSQL StatefulSet

```yaml
# k8s/postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - name: POSTGRES_DB
              value: mydb
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secrets
                  key: password
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1"
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 30
            periodSeconds: 10
  
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 20Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  clusterIP: None  # Headless service สำหรับ StatefulSet
```

### 8.3 Complete K8s Stack

```yaml
# k8s/kustomization.yaml - ใช้ Kustomize จัดการ manifests
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - namespace.yaml
  - configmap.yaml
  - postgres.yaml
  - redis.yaml
  - deployment.yaml
  - service.yaml
  - ingress.yaml
  - hpa.yaml

images:
  - name: ghcr.io/myorg/rust-api
    newTag: "1.0.0"

configMapGenerator:
  - name: rust-api-version
    literals:
      - VERSION=1.0.0
      - DEPLOYED_AT=2024-01-01T00:00:00Z
```

```bash
# Apply ทุก manifests
kubectl apply -k k8s/

# ดู status
kubectl get all -n production

# ดู pod logs
kubectl logs -f deployment/rust-api -n production

# Scale deployment
kubectl scale deployment/rust-api --replicas=5 -n production

# Rolling update
kubectl set image deployment/rust-api \
  rust-api=ghcr.io/myorg/rust-api:1.1.0 \
  -n production

# ดู rollout status
kubectl rollout status deployment/rust-api -n production

# Rollback
kubectl rollout undo deployment/rust-api -n production

# ดู rollout history
kubectl rollout history deployment/rust-api -n production
```

### 8.4 Monitoring กับ Prometheus

```rust
// src/metrics.rs - Expose metrics สำหรับ Prometheus
use actix_web::{web, HttpResponse};
use prometheus::{
    Counter, Histogram, HistogramVec, IntCounterVec,
    Registry, TextEncoder, Encoder,
};
use lazy_static::lazy_static;

lazy_static! {
    pub static ref HTTP_REQUESTS_TOTAL: IntCounterVec = IntCounterVec::new(
        prometheus::opts!("http_requests_total", "Total HTTP requests"),
        &["method", "path", "status"]
    ).unwrap();
    
    pub static ref HTTP_REQUEST_DURATION: HistogramVec = HistogramVec::new(
        prometheus::histogram_opts!(
            "http_request_duration_seconds",
            "HTTP request duration in seconds",
            vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0, 5.0]
        ),
        &["method", "path"]
    ).unwrap();
    
    pub static ref DB_QUERY_DURATION: HistogramVec = HistogramVec::new(
        prometheus::histogram_opts!(
            "db_query_duration_seconds",
            "Database query duration",
            vec![0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0]
        ),
        &["query_type"]
    ).unwrap();
}

pub fn register_metrics() {
    let registry = prometheus::default_registry();
    registry.register(Box::new(HTTP_REQUESTS_TOTAL.clone())).unwrap();
    registry.register(Box::new(HTTP_REQUEST_DURATION.clone())).unwrap();
    registry.register(Box::new(DB_QUERY_DURATION.clone())).unwrap();
}

pub async fn metrics_handler() -> HttpResponse {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();
    let mut buffer = vec![];
    encoder.encode(&metric_families, &mut buffer).unwrap();
    
    HttpResponse::Ok()
        .content_type("text/plain; version=0.0.4")
        .body(buffer)
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Deployment** - replicas, rolling updates, resource limits
2. **Service** - ClusterIP, NodePort, LoadBalancer
3. **ConfigMap & Secrets** - การจัดการ configuration
4. **Resource Limits** - LimitRange และ ResourceQuota
5. **HPA** - auto-scaling ตาม CPU/memory/custom metrics
6. **Ingress** - routing traffic เข้ามา
7. **Probes** - readiness, liveness, startup probes
8. **Complete stack** - namespace, DB, app, monitoring

---

[⬅️ Part 075: Deployment to Cloud](../part_075/README.md) | [➡️ Part 077: Database Backup and Recovery](../part_077/README.md)
