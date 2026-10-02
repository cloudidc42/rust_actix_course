# Part 097: Plugin System Architecture

## บทนำ (Introduction)

Plugin system ช่วยให้ application สามารถ extend functionality ได้โดยไม่ต้องแก้ไข core code ในบทนี้เราจะเรียนรู้วิธีสร้าง extensible application ด้วย dynamic loading, trait-based plugins, และ WASM-based sandboxed plugins

## 1. Dynamic Loading (libloading)

### โหลด shared library ในขณะ runtime

```toml
# Cargo.toml
[dependencies]
libloading = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
tokio = { version = "1", features = ["full"] }
uuid = { version = "1", features = ["v4"] }

# สำหรับ plugin library
[lib]
crate-type = ["cdylib"]  # dynamic library
```

```rust
use libloading::{Library, Symbol};
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use anyhow::{Context, Result};

// Plugin interface (ต้องเหมือนกันทั้ง host และ plugin)
#[repr(C)]
pub struct PluginInfo {
    pub name: *const std::ffi::c_char,
    pub version: *const std::ffi::c_char,
    pub description: *const std::ffi::c_char,
    pub api_version: u32,
}

// Function signatures ใน shared library
type CreatePlugin = unsafe fn() -> *mut dyn Plugin;
type DestroyPlugin = unsafe fn(*mut dyn Plugin);
type GetPluginInfo = unsafe fn() -> PluginInfo;

const API_VERSION: u32 = 1;

// Dynamic loader
struct DynamicPluginLoader {
    libraries: HashMap<String, Library>,
    plugins: HashMap<String, Box<dyn Plugin>>,
}

impl DynamicPluginLoader {
    fn new() -> Self {
        DynamicPluginLoader {
            libraries: HashMap::new(),
            plugins: HashMap::new(),
        }
    }
    
    unsafe fn load_plugin(&mut self, path: &Path) -> Result<String> {
        let lib = Library::new(path)
            .with_context(|| format!("Failed to load library: {}", path.display()))?;
        
        // Get plugin info
        let get_info: Symbol<GetPluginInfo> = lib.get(b"get_plugin_info")
            .context("Plugin missing get_plugin_info function")?;
        
        let info = get_info();
        
        // Check API version compatibility
        if info.api_version != API_VERSION {
            anyhow::bail!(
                "Plugin API version mismatch: expected {}, got {}",
                API_VERSION, info.api_version
            );
        }
        
        let name = std::ffi::CStr::from_ptr(info.name)
            .to_string_lossy()
            .into_owned();
        
        // Create plugin instance
        let create: Symbol<CreatePlugin> = lib.get(b"create_plugin")
            .context("Plugin missing create_plugin function")?;
        
        let plugin_ptr = create();
        let plugin = Box::from_raw(plugin_ptr);
        
        println!("Loaded plugin: {} v{}", name,
            std::ffi::CStr::from_ptr(info.version).to_string_lossy()
        );
        
        self.libraries.insert(name.clone(), lib);
        self.plugins.insert(name.clone(), plugin);
        
        Ok(name)
    }
    
    fn unload_plugin(&mut self, name: &str) {
        self.plugins.remove(name);
        // Library unloads when dropped
        self.libraries.remove(name);
        println!("Unloaded plugin: {}", name);
    }
    
    fn get_plugin(&self, name: &str) -> Option<&dyn Plugin> {
        self.plugins.get(name).map(|p| p.as_ref())
    }
}
```

## 2. Plugin Traits

### Define plugin interface

```rust
use std::collections::HashMap;
use serde_json::Value;

// Core plugin trait
pub trait Plugin: Send + Sync {
    fn name(&self) -> &str;
    fn version(&self) -> &str;
    fn description(&self) -> &str;
    
    fn initialize(&mut self, config: &PluginConfig) -> Result<(), PluginError>;
    fn shutdown(&mut self);
    
    fn capabilities(&self) -> Vec<String>;
}

// Specialized plugin traits
pub trait CommandPlugin: Plugin {
    fn commands(&self) -> Vec<CommandDef>;
    fn execute_command(&self, name: &str, args: &CommandArgs) -> Result<CommandResult, PluginError>;
}

pub trait DataProcessorPlugin: Plugin {
    fn process(&self, input: &[u8]) -> Result<Vec<u8>, PluginError>;
    fn supported_formats(&self) -> Vec<String>;
}

pub trait EventHandlerPlugin: Plugin {
    fn on_event(&self, event: &Event) -> Result<Option<Event>, PluginError>;
    fn subscribed_events(&self) -> Vec<String>;
}

pub trait AuthPlugin: Plugin {
    fn authenticate(&self, credentials: &Credentials) -> Result<AuthToken, PluginError>;
    fn authorize(&self, token: &AuthToken, resource: &str, action: &str) -> bool;
    fn refresh_token(&self, token: &AuthToken) -> Result<AuthToken, PluginError>;
}

// Supporting types
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
pub struct PluginConfig {
    pub settings: HashMap<String, Value>,
}

#[derive(Debug, Clone)]
pub struct CommandDef {
    pub name: String,
    pub description: String,
    pub args: Vec<ArgDef>,
}

#[derive(Debug, Clone)]
pub struct ArgDef {
    pub name: String,
    pub arg_type: String,
    pub required: bool,
    pub description: String,
}

#[derive(Debug, Clone)]
pub struct CommandArgs {
    pub values: HashMap<String, Value>,
}

#[derive(Debug, Clone)]
pub struct CommandResult {
    pub success: bool,
    pub output: Value,
    pub error: Option<String>,
}

#[derive(Debug, Clone)]
pub struct Event {
    pub id: String,
    pub event_type: String,
    pub payload: Value,
    pub timestamp: u64,
}

#[derive(Debug, Clone)]
pub struct Credentials {
    pub username: String,
    pub password: String,
    pub extra: HashMap<String, String>,
}

#[derive(Debug, Clone)]
pub struct AuthToken {
    pub token: String,
    pub expires_at: u64,
    pub claims: HashMap<String, Value>,
}

#[derive(Debug, thiserror::Error)]
pub enum PluginError {
    #[error("Initialization failed: {0}")]
    InitError(String),
    
    #[error("Command not found: {0}")]
    CommandNotFound(String),
    
    #[error("Invalid arguments: {0}")]
    InvalidArgs(String),
    
    #[error("Authentication failed: {0}")]
    AuthError(String),
    
    #[error("Plugin error: {0}")]
    General(String),
}
```

## 3. Plugin Registry

### จัดการ plugins

```rust
use std::sync::{Arc, RwLock};
use uuid::Uuid;

pub struct PluginRegistry {
    plugins: Arc<RwLock<HashMap<String, Arc<dyn Plugin>>>>,
    command_plugins: Arc<RwLock<HashMap<String, Arc<dyn CommandPlugin>>>>,
    event_plugins: Arc<RwLock<Vec<Arc<dyn EventHandlerPlugin>>>>,
    auth_plugin: Arc<RwLock<Option<Arc<dyn AuthPlugin>>>>,
}

impl PluginRegistry {
    pub fn new() -> Self {
        PluginRegistry {
            plugins: Arc::new(RwLock::new(HashMap::new())),
            command_plugins: Arc::new(RwLock::new(HashMap::new())),
            event_plugins: Arc::new(RwLock::new(Vec::new())),
            auth_plugin: Arc::new(RwLock::new(None)),
        }
    }
    
    pub fn register(&self, plugin: Arc<dyn Plugin>) -> Result<()> {
        let name = plugin.name().to_string();
        
        let mut plugins = self.plugins.write().unwrap();
        if plugins.contains_key(&name) {
            anyhow::bail!("Plugin '{}' is already registered", name);
        }
        
        plugins.insert(name.clone(), plugin.clone());
        println!("Registered plugin: {}", name);
        
        Ok(())
    }
    
    pub fn register_command_plugin(&self, plugin: Arc<dyn CommandPlugin>) -> Result<()> {
        let name = plugin.name().to_string();
        let mut plugins = self.command_plugins.write().unwrap();
        plugins.insert(name.clone(), plugin);
        println!("Registered command plugin: {}", name);
        Ok(())
    }
    
    pub fn register_event_plugin(&self, plugin: Arc<dyn EventHandlerPlugin>) {
        let mut plugins = self.event_plugins.write().unwrap();
        plugins.push(plugin);
    }
    
    pub fn set_auth_plugin(&self, plugin: Arc<dyn AuthPlugin>) {
        let mut auth = self.auth_plugin.write().unwrap();
        *auth = Some(plugin);
        println!("Auth plugin configured");
    }
    
    pub fn get_plugin(&self, name: &str) -> Option<Arc<dyn Plugin>> {
        self.plugins.read().unwrap().get(name).cloned()
    }
    
    pub fn execute_command(&self, plugin_name: &str, command: &str, args: CommandArgs) 
        -> Result<CommandResult, PluginError> 
    {
        let plugins = self.command_plugins.read().unwrap();
        let plugin = plugins.get(plugin_name)
            .ok_or_else(|| PluginError::CommandNotFound(plugin_name.to_string()))?;
        
        plugin.execute_command(command, &args)
    }
    
    pub fn dispatch_event(&self, event: Event) -> Vec<Event> {
        let plugins = self.event_plugins.read().unwrap();
        let mut results = Vec::new();
        
        for plugin in plugins.iter() {
            if plugin.subscribed_events().contains(&event.event_type) {
                if let Ok(Some(response)) = plugin.on_event(&event) {
                    results.push(response);
                }
            }
        }
        
        results
    }
    
    pub fn list_plugins(&self) -> Vec<PluginInfo> {
        let plugins = self.plugins.read().unwrap();
        plugins.values()
            .map(|p| PluginInfo {
                name: p.name().to_string(),
                version: p.version().to_string(),
                description: p.description().to_string(),
                capabilities: p.capabilities(),
            })
            .collect()
    }
}

#[derive(Debug)]
pub struct PluginInfo {
    pub name: String,
    pub version: String,
    pub description: String,
    pub capabilities: Vec<String>,
}

// Example plugin implementation
pub struct LoggingPlugin {
    config: Option<PluginConfig>,
    log_level: String,
}

impl LoggingPlugin {
    pub fn new() -> Self {
        LoggingPlugin {
            config: None,
            log_level: "info".to_string(),
        }
    }
}

impl Plugin for LoggingPlugin {
    fn name(&self) -> &str { "logging" }
    fn version(&self) -> &str { "1.0.0" }
    fn description(&self) -> &str { "Structured logging plugin" }
    
    fn initialize(&mut self, config: &PluginConfig) -> Result<(), PluginError> {
        if let Some(level) = config.settings.get("level").and_then(|v| v.as_str()) {
            self.log_level = level.to_string();
        }
        self.config = Some(config.clone());
        println!("Logging plugin initialized (level: {})", self.log_level);
        Ok(())
    }
    
    fn shutdown(&mut self) {
        println!("Logging plugin shutdown");
    }
    
    fn capabilities(&self) -> Vec<String> {
        vec!["logging".to_string(), "metrics".to_string()]
    }
}

impl EventHandlerPlugin for LoggingPlugin {
    fn subscribed_events(&self) -> Vec<String> {
        vec!["*".to_string()] // subscribe to all events
    }
    
    fn on_event(&self, event: &Event) -> Result<Option<Event>, PluginError> {
        println!("[{}] Event: {} - {}", 
            self.log_level,
            event.event_type,
            serde_json::to_string(&event.payload).unwrap_or_default()
        );
        Ok(None)
    }
}
```

## 4. Hot Reloading Plugins

### ตรวจจับการเปลี่ยนแปลง library

```rust
use std::time::{Duration, SystemTime};
use tokio::fs;

struct HotReloadWatcher {
    plugin_dir: PathBuf,
    loaded_plugins: HashMap<PathBuf, (SystemTime, String)>,
    registry: Arc<PluginRegistry>,
}

impl HotReloadWatcher {
    fn new(plugin_dir: PathBuf, registry: Arc<PluginRegistry>) -> Self {
        HotReloadWatcher {
            plugin_dir,
            loaded_plugins: HashMap::new(),
            registry,
        }
    }
    
    async fn watch(&mut self) {
        println!("Watching {} for plugin changes...", self.plugin_dir.display());
        
        let mut interval = tokio::time::interval(Duration::from_secs(2));
        
        loop {
            interval.tick().await;
            
            if let Err(e) = self.check_for_changes().await {
                eprintln!("Error checking plugins: {}", e);
            }
        }
    }
    
    async fn check_for_changes(&mut self) -> Result<()> {
        let mut dir = tokio::fs::read_dir(&self.plugin_dir).await?;
        
        while let Some(entry) = dir.next_entry().await? {
            let path = entry.path();
            
            // Only watch .so/.dll/.dylib files
            let ext = path.extension().and_then(|e| e.to_str()).unwrap_or("");
            if !matches!(ext, "so" | "dll" | "dylib") {
                continue;
            }
            
            let metadata = tokio::fs::metadata(&path).await?;
            let modified = metadata.modified()?;
            
            if let Some((last_modified, _)) = self.loaded_plugins.get(&path) {
                if modified > *last_modified {
                    println!("Plugin changed, reloading: {}", path.display());
                    self.reload_plugin(&path).await?;
                }
            } else {
                println!("New plugin found: {}", path.display());
                self.load_plugin(&path).await?;
            }
        }
        
        Ok(())
    }
    
    async fn load_plugin(&mut self, path: &Path) -> Result<()> {
        let metadata = tokio::fs::metadata(path).await?;
        let modified = metadata.modified()?;
        
        // In a real implementation, we would load the plugin here
        let name = path.file_stem()
            .unwrap_or_default()
            .to_string_lossy()
            .to_string();
        
        println!("Loaded plugin: {}", name);
        self.loaded_plugins.insert(path.to_path_buf(), (modified, name));
        
        Ok(())
    }
    
    async fn reload_plugin(&mut self, path: &Path) -> Result<()> {
        // Unload old version
        if let Some((_, name)) = self.loaded_plugins.get(path) {
            println!("Unloading old version of: {}", name);
        }
        
        // Load new version
        self.load_plugin(path).await
    }
}
```

## 5. WASM-based Plugins

### Sandboxed plugins ด้วย WebAssembly

```rust
// ใช้ wasmtime หรือ wasmer สำหรับ WASM plugins
// Cargo.toml: wasmtime = "15"

use std::collections::HashMap;

// WASM plugin interface
struct WasmPlugin {
    name: String,
    wasm_bytes: Vec<u8>,
    // engine: wasmtime::Engine,
    // instance: wasmtime::Instance,
}

impl WasmPlugin {
    fn load(path: &Path) -> Result<Self> {
        let wasm_bytes = std::fs::read(path)
            .with_context(|| format!("Failed to read WASM file: {}", path.display()))?;
        
        let name = path.file_stem()
            .unwrap_or_default()
            .to_string_lossy()
            .to_string();
        
        // Validate WASM
        wasmparser::validate(&wasm_bytes)
            .context("Invalid WASM binary")?;
        
        println!("Loaded WASM plugin: {} ({} bytes)", name, wasm_bytes.len());
        
        Ok(WasmPlugin {
            name,
            wasm_bytes,
        })
    }
    
    fn call_function(&self, func_name: &str, args: &[i32]) -> Result<Vec<i32>> {
        // In real implementation using wasmtime:
        // let engine = wasmtime::Engine::default();
        // let module = wasmtime::Module::new(&engine, &self.wasm_bytes)?;
        // let linker = wasmtime::Linker::new(&engine);
        // let mut store = wasmtime::Store::new(&engine, ());
        // let instance = linker.instantiate(&mut store, &module)?;
        // 
        // let func = instance.get_typed_func::<(i32, i32), i32>(&mut store, func_name)?;
        // let result = func.call(&mut store, (args[0], args[1]))?;
        
        // Mock implementation
        println!("Calling WASM function: {}({:?})", func_name, args);
        Ok(vec![42]) // mock result
    }
}

// Plugin สำหรับ WASM
struct WasmPluginManager {
    plugins: HashMap<String, WasmPlugin>,
    sandbox_limits: SandboxLimits,
}

#[derive(Debug, Clone)]
struct SandboxLimits {
    max_memory_mb: u32,
    max_execution_ms: u64,
    allowed_imports: Vec<String>,
}

impl Default for SandboxLimits {
    fn default() -> Self {
        SandboxLimits {
            max_memory_mb: 64,
            max_execution_ms: 5000,
            allowed_imports: vec![
                "console_log".to_string(),
                "get_config".to_string(),
                "emit_event".to_string(),
            ],
        }
    }
}

impl WasmPluginManager {
    fn new() -> Self {
        WasmPluginManager {
            plugins: HashMap::new(),
            sandbox_limits: SandboxLimits::default(),
        }
    }
    
    fn load_plugin(&mut self, path: &Path) -> Result<()> {
        let plugin = WasmPlugin::load(path)?;
        self.plugins.insert(plugin.name.clone(), plugin);
        Ok(())
    }
    
    fn execute(&self, plugin_name: &str, func: &str, input: &[u8]) -> Result<Vec<u8>> {
        let plugin = self.plugins.get(plugin_name)
            .ok_or_else(|| anyhow::anyhow!("Plugin not found: {}", plugin_name))?;
        
        // Execute with timeout
        let timeout = Duration::from_millis(self.sandbox_limits.max_execution_ms);
        
        // Mock execution
        println!("Executing WASM plugin '{}' function '{}'", plugin_name, func);
        Ok(b"result".to_vec())
    }
}
```

## 6. Practical: Extensible App with Plugins

### Complete extensible application

```rust
use std::sync::Arc;
use tokio;

// Application core
struct Application {
    registry: Arc<PluginRegistry>,
    config: AppConfig,
}

#[derive(Debug, serde::Deserialize)]
struct AppConfig {
    plugin_dir: String,
    enabled_plugins: Vec<String>,
}

impl Default for AppConfig {
    fn default() -> Self {
        AppConfig {
            plugin_dir: "./plugins".to_string(),
            enabled_plugins: vec![
                "logging".to_string(),
                "auth".to_string(),
            ],
        }
    }
}

impl Application {
    fn new(config: AppConfig) -> Self {
        Application {
            registry: Arc::new(PluginRegistry::new()),
            config,
        }
    }
    
    async fn load_plugins(&self) -> Result<()> {
        println!("Loading plugins from: {}", self.config.plugin_dir);
        
        // Load built-in plugins
        let logging = Arc::new(LoggingPlugin::new());
        self.registry.register_event_plugin(logging.clone());
        
        let auth = Arc::new(BasicAuthPlugin::new());
        self.registry.set_auth_plugin(auth);
        
        let commands = Arc::new(DatabaseCommandPlugin::new());
        self.registry.register_command_plugin(commands)?;
        
        println!("Loaded {} plugins", 3);
        Ok(())
    }
    
    async fn run(&self) -> Result<()> {
        println!("Application started");
        
        // List loaded plugins
        let plugins = self.registry.list_plugins();
        println!("\nLoaded plugins:");
        for p in &plugins {
            println!("  - {} v{}: {}", p.name, p.version, p.description);
            println!("    Capabilities: {:?}", p.capabilities);
        }
        
        // Dispatch some events
        let event = Event {
            id: uuid::Uuid::new_v4().to_string(),
            event_type: "user.login".to_string(),
            payload: serde_json::json!({
                "user_id": 123,
                "ip": "192.168.1.1"
            }),
            timestamp: 1700000000,
        };
        
        println!("\nDispatching event: {}", event.event_type);
        let responses = self.registry.dispatch_event(event);
        println!("Got {} responses", responses.len());
        
        // Execute command
        let args = CommandArgs {
            values: {
                let mut m = HashMap::new();
                m.insert("limit".to_string(), serde_json::Value::Number(10.into()));
                m
            },
        };
        
        println!("\nExecuting command...");
        match self.registry.execute_command("database", "list_users", args) {
            Ok(result) => println!("Command result: {:?}", result.output),
            Err(e) => println!("Command error: {}", e),
        }
        
        Ok(())
    }
}

// Built-in auth plugin
struct BasicAuthPlugin;

impl BasicAuthPlugin {
    fn new() -> Self { BasicAuthPlugin }
}

impl Plugin for BasicAuthPlugin {
    fn name(&self) -> &str { "auth" }
    fn version(&self) -> &str { "1.0.0" }
    fn description(&self) -> &str { "Basic authentication plugin" }
    
    fn initialize(&mut self, _config: &PluginConfig) -> Result<(), PluginError> {
        println!("Auth plugin initialized");
        Ok(())
    }
    
    fn shutdown(&mut self) {}
    
    fn capabilities(&self) -> Vec<String> {
        vec!["authentication".to_string(), "authorization".to_string()]
    }
}

impl AuthPlugin for BasicAuthPlugin {
    fn authenticate(&self, credentials: &Credentials) -> Result<AuthToken, PluginError> {
        // Mock authentication
        if credentials.username == "admin" && credentials.password == "secret" {
            Ok(AuthToken {
                token: "mock-jwt-token".to_string(),
                expires_at: 1800000000,
                claims: {
                    let mut m = HashMap::new();
                    m.insert("role".to_string(), serde_json::Value::String("admin".to_string()));
                    m
                },
            })
        } else {
            Err(PluginError::AuthError("Invalid credentials".to_string()))
        }
    }
    
    fn authorize(&self, token: &AuthToken, resource: &str, action: &str) -> bool {
        // Mock authorization
        token.claims.get("role")
            .and_then(|r| r.as_str())
            .map(|r| r == "admin")
            .unwrap_or(false)
    }
    
    fn refresh_token(&self, token: &AuthToken) -> Result<AuthToken, PluginError> {
        Ok(token.clone())
    }
}

// Database command plugin
struct DatabaseCommandPlugin {
    commands: Vec<CommandDef>,
}

impl DatabaseCommandPlugin {
    fn new() -> Self {
        DatabaseCommandPlugin {
            commands: vec![
                CommandDef {
                    name: "list_users".to_string(),
                    description: "List all users".to_string(),
                    args: vec![
                        ArgDef {
                            name: "limit".to_string(),
                            arg_type: "integer".to_string(),
                            required: false,
                            description: "Max users to return".to_string(),
                        },
                    ],
                },
                CommandDef {
                    name: "create_user".to_string(),
                    description: "Create a new user".to_string(),
                    args: vec![
                        ArgDef {
                            name: "username".to_string(),
                            arg_type: "string".to_string(),
                            required: true,
                            description: "Username".to_string(),
                        },
                    ],
                },
            ],
        }
    }
}

impl Plugin for DatabaseCommandPlugin {
    fn name(&self) -> &str { "database" }
    fn version(&self) -> &str { "1.0.0" }
    fn description(&self) -> &str { "Database operations plugin" }
    
    fn initialize(&mut self, _config: &PluginConfig) -> Result<(), PluginError> {
        println!("Database plugin initialized");
        Ok(())
    }
    
    fn shutdown(&mut self) {}
    
    fn capabilities(&self) -> Vec<String> {
        vec!["database".to_string(), "crud".to_string()]
    }
}

impl CommandPlugin for DatabaseCommandPlugin {
    fn commands(&self) -> Vec<CommandDef> {
        self.commands.clone()
    }
    
    fn execute_command(&self, name: &str, args: &CommandArgs) -> Result<CommandResult, PluginError> {
        match name {
            "list_users" => {
                let limit = args.values.get("limit")
                    .and_then(|v| v.as_u64())
                    .unwrap_or(10);
                
                let users: Vec<serde_json::Value> = (1..=limit)
                    .map(|i| serde_json::json!({
                        "id": i,
                        "username": format!("user_{}", i),
                        "email": format!("user{}@example.com", i)
                    }))
                    .collect();
                
                Ok(CommandResult {
                    success: true,
                    output: serde_json::Value::Array(users),
                    error: None,
                })
            }
            "create_user" => {
                let username = args.values.get("username")
                    .and_then(|v| v.as_str())
                    .ok_or_else(|| PluginError::InvalidArgs("username is required".to_string()))?;
                
                Ok(CommandResult {
                    success: true,
                    output: serde_json::json!({
                        "id": 999,
                        "username": username,
                        "created": true
                    }),
                    error: None,
                })
            }
            _ => Err(PluginError::CommandNotFound(name.to_string())),
        }
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let config = AppConfig::default();
    let app = Application::new(config);
    
    app.load_plugins().await?;
    app.run().await?;
    
    Ok(())
}
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **Dynamic Loading**: ใช้ libloading สำหรับ load shared libraries ใน runtime
- **Plugin Traits**: design traits สำหรับ plugin interface
- **Plugin Registry**: จัดการ plugins และ dispatch events/commands
- **Hot Reloading**: ตรวจจับ plugin changes และ reload อัตโนมัติ
- **WASM Plugins**: sandboxed plugins ด้วย WebAssembly
- **Security**: sandboxing และ limits สำหรับ plugins
- **Complete App**: ตัวอย่าง extensible application

---

[← Part 096](../part_096/README.md) | [Part 098 →](../part_098/README.md)
