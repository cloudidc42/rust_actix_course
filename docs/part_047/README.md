# Part 047: TLS and HTTPS 🔐

## 🎯 เป้าหมายของ Part นี้

- ใช้ rustls สำหรับ TLS
- Setup TLS certificate
- Self-signed certs สำหรับ development
- HTTPS redirect middleware
- HSTS header
- actix-web กับ TLS
- Certificate renewal
- Environment-based TLS config

---

## 1. TLS Concepts

**TLS (Transport Layer Security)** เป็น protocol ที่เข้ารหัส traffic ระหว่าง client และ server

**ทำไมต้อง HTTPS:**
- เข้ารหัส data ระหว่างส่ง
- ป้องกัน man-in-the-middle attacks
- ยืนยัน identity ของ server
- Required สำหรับ HTTP/2, cookies Secure flag

---

## 2. Setup

### 2.1 Cargo.toml

```toml
[package]
name = "tls-https"
version = "0.1.0"
edition = "2021"

[dependencies]
actix-web = { version = "4", features = ["rustls-0_22"] }
actix-web-lab = "0.20"
rustls = "0.22"
rustls-pemfile = "2"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
log = "0.4"
env_logger = "0.10"
dotenv = "0.15"
openssl = { version = "0.10", features = ["v102", "v110"] }
```

---

## 3. สร้าง Self-signed Certificate สำหรับ Development

### 3.1 ใช้ OpenSSL command line

```bash
# สร้าง private key
openssl genrsa -out server.key 2048

# สร้าง certificate signing request (CSR)
openssl req -new -key server.key -out server.csr \
  -subj "/C=TH/ST=Bangkok/L=Bangkok/O=MyApp/CN=localhost"

# สร้าง self-signed certificate (valid 365 วัน)
openssl x509 -req -days 365 -in server.csr \
  -signkey server.key -out server.crt

# หรือสร้างในคำสั่งเดียว
openssl req -x509 -newkey rsa:2048 -keyout server.key \
  -out server.crt -days 365 -nodes \
  -subj "/C=TH/ST=Bangkok/L=Bangkok/O=MyApp/CN=localhost"
```

### 3.2 สร้าง Certificate ด้วย Rust

```rust
// src/cert_gen.rs
use std::path::Path;
use std::fs;

/// สร้าง self-signed certificate สำหรับ development
pub fn generate_self_signed_cert(
    cert_path: &Path,
    key_path: &Path,
) -> anyhow::Result<()> {
    // ตรวจสอบว่า cert มีอยู่แล้วหรือไม่
    if cert_path.exists() && key_path.exists() {
        log::info!("Certificates already exist, skipping generation");
        return Ok(());
    }
    
    use openssl::{
        asn1::Asn1Time,
        bn::{BigNum, MsbOption},
        hash::MessageDigest,
        pkey::PKey,
        rsa::Rsa,
        x509::{
            extension::{BasicConstraints, KeyUsage, SubjectAlternativeName},
            X509Builder, X509NameBuilder,
        },
    };
    
    // สร้าง private key
    let rsa = Rsa::generate(2048)?;
    let pkey = PKey::from_rsa(rsa)?;
    
    // สร้าง certificate
    let mut builder = X509Builder::new()?;
    builder.set_version(2)?;
    
    let serial = BigNum::from_u32(1)?;
    let serial = serial.to_asn1_integer()?;
    builder.set_serial_number(&serial)?;
    
    // Set subject
    let mut name_builder = X509NameBuilder::new()?;
    name_builder.append_entry_by_text("C", "TH")?;
    name_builder.append_entry_by_text("ST", "Bangkok")?;
    name_builder.append_entry_by_text("L", "Bangkok")?;
    name_builder.append_entry_by_text("O", "Development")?;
    name_builder.append_entry_by_text("CN", "localhost")?;
    let name = name_builder.build();
    
    builder.set_subject_name(&name)?;
    builder.set_issuer_name(&name)?;
    builder.set_pubkey(&pkey)?;
    
    // Set validity
    let not_before = Asn1Time::days_from_now(0)?;
    let not_after = Asn1Time::days_from_now(365)?;
    builder.set_not_before(&not_before)?;
    builder.set_not_after(&not_after)?;
    
    // Extensions
    let basic_constraints = BasicConstraints::new().build()?;
    builder.append_extension(basic_constraints)?;
    
    let key_usage = KeyUsage::new()
        .digital_signature()
        .key_encipherment()
        .build()?;
    builder.append_extension(key_usage)?;
    
    // SAN (Subject Alternative Names) สำหรับ localhost
    let san = SubjectAlternativeName::new()
        .dns("localhost")
        .ip("127.0.0.1")
        .build(&builder.x509v3_context(None, None))?;
    builder.append_extension(san)?;
    
    // Sign certificate
    builder.sign(&pkey, MessageDigest::sha256())?;
    
    let cert = builder.build();
    
    // เขียน files
    fs::write(cert_path, cert.to_pem()?)?;
    fs::write(key_path, pkey.private_key_to_pem_pkcs8()?)?;
    
    log::info!("Generated self-signed certificate at {:?}", cert_path);
    
    Ok(())
}
```

---

## 4. TLS Configuration

```rust
// src/tls_config.rs
use rustls::{ServerConfig, Certificate, PrivateKey};
use rustls_pemfile::{certs, pkcs8_private_keys, rsa_private_keys};
use std::fs::File;
use std::io::BufReader;
use std::path::Path;

#[derive(Debug, Clone)]
pub struct TlsSettings {
    pub cert_path: String,
    pub key_path: String,
    pub min_version: TlsVersion,
}

#[derive(Debug, Clone, PartialEq)]
pub enum TlsVersion {
    Tls12,
    Tls13,
}

impl TlsSettings {
    pub fn from_env() -> Option<Self> {
        let cert_path = std::env::var("TLS_CERT_PATH").ok()?;
        let key_path = std::env::var("TLS_KEY_PATH").ok()?;
        
        let min_version = std::env::var("TLS_MIN_VERSION")
            .map(|v| if v == "1.3" { TlsVersion::Tls13 } else { TlsVersion::Tls12 })
            .unwrap_or(TlsVersion::Tls12);
        
        Some(Self { cert_path, key_path, min_version })
    }
    
    pub fn dev_default() -> Self {
        Self {
            cert_path: "certs/server.crt".to_string(),
            key_path: "certs/server.key".to_string(),
            min_version: TlsVersion::Tls12,
        }
    }
}

/// สร้าง rustls ServerConfig
pub fn create_tls_config(settings: &TlsSettings) -> anyhow::Result<ServerConfig> {
    // โหลด certificates
    let cert_file = File::open(&settings.cert_path)
        .map_err(|e| anyhow::anyhow!("Cannot open cert file {}: {}", settings.cert_path, e))?;
    let key_file = File::open(&settings.key_path)
        .map_err(|e| anyhow::anyhow!("Cannot open key file {}: {}", settings.key_path, e))?;
    
    let mut cert_reader = BufReader::new(cert_file);
    let mut key_reader = BufReader::new(key_file);
    
    // Parse certificates
    let cert_chain: Vec<Certificate> = certs(&mut cert_reader)
        .map_err(|e| anyhow::anyhow!("Failed to parse certificates: {}", e))?
        .into_iter()
        .map(Certificate)
        .collect();
    
    if cert_chain.is_empty() {
        return Err(anyhow::anyhow!("No certificates found in cert file"));
    }
    
    // Parse private key (ลอง PKCS8 ก่อน แล้วค่อย RSA)
    let private_key = pkcs8_private_keys(&mut key_reader)
        .map_err(|e| anyhow::anyhow!("Failed to parse PKCS8 key: {}", e))?
        .into_iter()
        .next()
        .map(PrivateKey);
    
    let private_key = match private_key {
        Some(key) => key,
        None => {
            // ลอง RSA key
            let mut key_reader = BufReader::new(
                File::open(&settings.key_path)?
            );
            rsa_private_keys(&mut key_reader)
                .map_err(|e| anyhow::anyhow!("Failed to parse RSA key: {}", e))?
                .into_iter()
                .next()
                .map(PrivateKey)
                .ok_or_else(|| anyhow::anyhow!("No private key found"))?
        }
    };
    
    // สร้าง TLS config
    let config = match settings.min_version {
        TlsVersion::Tls13 => {
            ServerConfig::builder()
                .with_safe_defaults()
                .with_no_client_auth()
                .with_single_cert(cert_chain, private_key)?
        },
        TlsVersion::Tls12 => {
            ServerConfig::builder()
                .with_safe_defaults()
                .with_no_client_auth()
                .with_single_cert(cert_chain, private_key)?
        }
    };
    
    Ok(config)
}
```

---

## 5. HTTPS Redirect Middleware

```rust
// src/middleware/https_redirect.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error, HttpResponse,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;

/// Middleware redirect HTTP → HTTPS
pub struct HttpsRedirect {
    https_port: u16,
}

impl HttpsRedirect {
    pub fn new(https_port: u16) -> Self {
        Self { https_port }
    }
}

impl<S, B> Transform<S, ServiceRequest> for HttpsRedirect
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = HttpsRedirectMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(HttpsRedirectMiddleware {
            service: Rc::new(service),
            https_port: self.https_port,
        }))
    }
}

pub struct HttpsRedirectMiddleware<S> {
    service: Rc<S>,
    https_port: u16,
}

impl<S, B> Service<ServiceRequest> for HttpsRedirectMiddleware<S>
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
        let https_port = self.https_port;
        
        Box::pin(async move {
            // ตรวจสอบว่าเป็น HTTPS หรือไม่
            let is_https = req.connection_info().scheme() == "https"
                || req.headers().get("X-Forwarded-Proto")
                    .and_then(|h| h.to_str().ok())
                    .map(|s| s == "https")
                    .unwrap_or(false);
            
            if is_https {
                return service.call(req).await;
            }
            
            // สร้าง HTTPS URL
            let host = req.connection_info().host().to_string();
            let path = req.uri().path_and_query()
                .map(|pq| pq.as_str())
                .unwrap_or("/");
            
            // ปรับ port
            let redirect_host = if https_port == 443 {
                // ตัด port ออกถ้าเป็น standard
                host.split(':').next().unwrap_or(&host).to_string()
            } else {
                let base_host = host.split(':').next().unwrap_or(&host);
                format!("{}:{}", base_host, https_port)
            };
            
            let redirect_url = format!("https://{}{}", redirect_host, path);
            
            let response = req.into_response(
                HttpResponse::PermanentRedirect()
                    .append_header(("Location", redirect_url))
                    .finish()
                    .map_into_boxed_body()
            );
            
            Ok(response)
        })
    }
}
```

---

## 6. Security Headers

```rust
// src/middleware/security_headers.rs
use actix_web::{
    dev::{forward_ready, Service, ServiceRequest, ServiceResponse, Transform},
    Error,
};
use futures::future::{ready, Ready, LocalBoxFuture};
use std::rc::Rc;

pub struct SecurityHeaders {
    hsts_max_age: u64,
    include_subdomains: bool,
}

impl SecurityHeaders {
    pub fn new() -> Self {
        Self {
            hsts_max_age: 31536000, // 1 year
            include_subdomains: true,
        }
    }
    
    pub fn with_max_age(mut self, seconds: u64) -> Self {
        self.hsts_max_age = seconds;
        self
    }
}

impl<S, B> Transform<S, ServiceRequest> for SecurityHeaders
where
    S: Service<ServiceRequest, Response = ServiceResponse<B>, Error = Error> + 'static,
    B: 'static,
{
    type Response = ServiceResponse<B>;
    type Error = Error;
    type Transform = SecurityHeadersMiddleware<S>;
    type InitError = ();
    type Future = Ready<Result<Self::Transform, Self::InitError>>;
    
    fn new_transform(&self, service: S) -> Self::Future {
        ready(Ok(SecurityHeadersMiddleware {
            service: Rc::new(service),
            hsts_max_age: self.hsts_max_age,
            include_subdomains: self.include_subdomains,
        }))
    }
}

pub struct SecurityHeadersMiddleware<S> {
    service: Rc<S>,
    hsts_max_age: u64,
    include_subdomains: bool,
}

impl<S, B> Service<ServiceRequest> for SecurityHeadersMiddleware<S>
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
        let hsts_max_age = self.hsts_max_age;
        let include_subdomains = self.include_subdomains;
        
        Box::pin(async move {
            let mut response = service.call(req).await?;
            let headers = response.headers_mut();
            
            // HTTP Strict Transport Security (HSTS)
            let hsts_value = if include_subdomains {
                format!("max-age={}; includeSubDomains; preload", hsts_max_age)
            } else {
                format!("max-age={}", hsts_max_age)
            };
            headers.insert(
                actix_web::http::header::HeaderName::from_static("strict-transport-security"),
                actix_web::http::header::HeaderValue::from_str(&hsts_value).unwrap(),
            );
            
            // X-Content-Type-Options
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-content-type-options"),
                actix_web::http::header::HeaderValue::from_static("nosniff"),
            );
            
            // X-Frame-Options
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-frame-options"),
                actix_web::http::header::HeaderValue::from_static("DENY"),
            );
            
            // X-XSS-Protection
            headers.insert(
                actix_web::http::header::HeaderName::from_static("x-xss-protection"),
                actix_web::http::header::HeaderValue::from_static("1; mode=block"),
            );
            
            // Referrer-Policy
            headers.insert(
                actix_web::http::header::HeaderName::from_static("referrer-policy"),
                actix_web::http::header::HeaderValue::from_static("strict-origin-when-cross-origin"),
            );
            
            // Content-Security-Policy (ปรับตาม needs)
            headers.insert(
                actix_web::http::header::HeaderName::from_static("content-security-policy"),
                actix_web::http::header::HeaderValue::from_static(
                    "default-src 'self'; script-src 'self'; style-src 'self'"
                ),
            );
            
            Ok(response)
        })
    }
}
```

---

## 7. Main Application พร้อม TLS

```rust
// src/main.rs
use actix_web::{web, App, HttpServer, middleware, HttpResponse};
use std::net::TcpListener;
use dotenv::dotenv;

mod cert_gen;
mod tls_config;
mod middleware {
    pub mod https_redirect;
    pub mod security_headers;
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init();
    
    let env = std::env::var("APP_ENV").unwrap_or_else(|_| "development".to_string());
    
    // สร้าง app factory
    let app_factory = || {
        App::new()
            .wrap(middleware::Logger::default())
            .wrap(middleware::security_headers::SecurityHeaders::new())
            .route("/", web::get().to(index))
            .route("/health", web::get().to(health))
    };
    
    if env == "production" {
        // Production: HTTPS only
        let tls_settings = tls_config::TlsSettings::from_env()
            .expect("TLS settings required in production");
        
        let tls_config = tls_config::create_tls_config(&tls_settings)
            .expect("Failed to create TLS config");
        
        log::info!("Starting HTTPS server on 0.0.0.0:443");
        
        // HTTP server สำหรับ redirect เท่านั้น
        let http_server = HttpServer::new(|| {
            App::new()
                .wrap(middleware::https_redirect::HttpsRedirect::new(443))
                .route("/{_:.*}", web::get().to(|| async {
                    HttpResponse::PermanentRedirect().finish()
                }))
        })
        .bind("0.0.0.0:80")?;
        
        // HTTPS server
        let https_server = HttpServer::new(app_factory)
            .bind_rustls_0_22("0.0.0.0:443", tls_config)?;
        
        tokio::try_join!(
            http_server.run(),
            https_server.run(),
        )?;
        
    } else {
        // Development: ใช้ self-signed cert
        let cert_dir = std::path::Path::new("certs");
        std::fs::create_dir_all(cert_dir)?;
        
        let cert_path = cert_dir.join("server.crt");
        let key_path = cert_dir.join("server.key");
        
        // สร้าง self-signed cert ถ้ายังไม่มี
        cert_gen::generate_self_signed_cert(&cert_path, &key_path)
            .expect("Failed to generate certificates");
        
        let tls_settings = tls_config::TlsSettings {
            cert_path: cert_path.to_str().unwrap().to_string(),
            key_path: key_path.to_str().unwrap().to_string(),
            min_version: tls_config::TlsVersion::Tls12,
        };
        
        let tls_config = tls_config::create_tls_config(&tls_settings)
            .expect("Failed to create TLS config");
        
        log::info!("Starting development HTTPS server on https://localhost:8443");
        log::info!("Note: Self-signed cert will show browser warning - that's OK for dev");
        
        HttpServer::new(app_factory)
            .bind_rustls_0_22("0.0.0.0:8443", tls_config)?
            .bind("0.0.0.0:8080")? // HTTP สำหรับ development ด้วย
            .run()
            .await?;
    }
    
    Ok(())
}

async fn index() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({
        "message": "Hello from HTTPS server!",
        "secure": true
    }))
}

async fn health() -> HttpResponse {
    HttpResponse::Ok().json(serde_json::json!({"status": "ok"}))
}
```

---

## 8. Let's Encrypt Certificate (Production)

```bash
# ติดตั้ง certbot
sudo apt-get install certbot

# สร้าง certificate สำหรับ domain
sudo certbot certonly --standalone \
  -d yourdomain.com \
  -d www.yourdomain.com \
  --email admin@yourdomain.com \
  --agree-tos

# Certificates จะอยู่ที่:
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem (cert)
# /etc/letsencrypt/live/yourdomain.com/privkey.pem (key)

# Auto-renewal (เพิ่มใน crontab)
# 0 2 * * * /usr/bin/certbot renew --quiet --post-hook "systemctl reload myapp"
```

### 8.1 Certificate Renewal Handler

```rust
// src/cert_renewal.rs
use std::path::Path;
use chrono::{Utc, Duration};

/// ตรวจสอบอายุของ certificate
pub fn check_cert_expiry(cert_path: &Path) -> anyhow::Result<i64> {
    use openssl::x509::X509;
    
    let cert_data = std::fs::read(cert_path)?;
    let cert = X509::from_pem(&cert_data)?;
    
    let not_after = cert.not_after();
    let not_after_str = not_after.to_string();
    
    // Parse the date (openssl format: "Jan  1 00:00:00 2025 GMT")
    // Days until expiry
    let days_remaining = (chrono::NaiveDate::parse_from_str(
        &not_after_str[4..15].trim(),
        "%b %d %Y",
    )
    .unwrap_or_else(|_| (Utc::now() + Duration::days(30)).date_naive())
    - Utc::now().date_naive())
    .num_days();
    
    Ok(days_remaining)
}

/// Warning ถ้า cert จะหมดอายุใน 30 วัน
pub async fn certificate_expiry_monitor(cert_path: String) {
    loop {
        match check_cert_expiry(Path::new(&cert_path)) {
            Ok(days) => {
                if days <= 30 {
                    log::warn!("Certificate expires in {} days! Please renew.", days);
                } else {
                    log::info!("Certificate valid for {} more days", days);
                }
            },
            Err(e) => log::error!("Failed to check certificate expiry: {}", e),
        }
        
        // Check ทุก 24 ชั่วโมง
        tokio::time::sleep(tokio::time::Duration::from_secs(86400)).await;
    }
}
```

---

## 9. Environment-based TLS Config

```rust
// src/config.rs
#[derive(Debug, Clone)]
pub enum AppEnvironment {
    Development,
    Staging,
    Production,
}

#[derive(Debug, Clone)]
pub struct AppConfig {
    pub env: AppEnvironment,
    pub http_port: u16,
    pub https_port: u16,
    pub tls_cert: Option<String>,
    pub tls_key: Option<String>,
    pub force_https: bool,
    pub hsts_enabled: bool,
}

impl AppConfig {
    pub fn from_env() -> Self {
        let env_str = std::env::var("APP_ENV").unwrap_or_else(|_| "development".to_string());
        let env = match env_str.as_str() {
            "production" => AppEnvironment::Production,
            "staging" => AppEnvironment::Staging,
            _ => AppEnvironment::Development,
        };
        
        let (force_https, hsts_enabled) = match env {
            AppEnvironment::Production => (true, true),
            AppEnvironment::Staging => (true, false),
            AppEnvironment::Development => (false, false),
        };
        
        Self {
            env,
            http_port: std::env::var("HTTP_PORT")
                .ok().and_then(|p| p.parse().ok()).unwrap_or(8080),
            https_port: std::env::var("HTTPS_PORT")
                .ok().and_then(|p| p.parse().ok()).unwrap_or(8443),
            tls_cert: std::env::var("TLS_CERT_PATH").ok(),
            tls_key: std::env::var("TLS_KEY_PATH").ok(),
            force_https,
            hsts_enabled,
        }
    }
    
    pub fn tls_enabled(&self) -> bool {
        self.tls_cert.is_some() && self.tls_key.is_some()
    }
}
```

---

## 10. Testing TLS

```bash
# ทดสอบ TLS connection
openssl s_client -connect localhost:8443 -tls1_2

# ตรวจสอบ certificate
openssl s_client -connect localhost:8443 2>/dev/null | openssl x509 -noout -text

# curl กับ self-signed cert (ข้าม verification สำหรับ dev)
curl -k https://localhost:8443/health

# curl กับ real cert
curl https://yourdomain.com/health
```

---

## 11. สรุปสิ่งที่เรียนรู้

✅ rustls crate สำหรับ TLS  
✅ สร้าง self-signed certificate ด้วย openssl  
✅ สร้าง certificate ด้วย Rust code  
✅ rustls ServerConfig  
✅ HTTPS redirect middleware  
✅ HSTS header  
✅ Security headers (CSP, X-Frame-Options, etc.)  
✅ Let's Encrypt certificate setup  
✅ Certificate expiry monitoring  
✅ Environment-based TLS configuration  

---

*[← Part 046: API Keys Management](../part_046/README.md) | [Part 048: Input Validation Advanced →](../part_048/README.md)*
