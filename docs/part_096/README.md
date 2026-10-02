# Part 096: Building CLI Tools

## บทนำ (Introduction)

Rust เป็นภาษาที่ยอดเยี่ยมสำหรับการสร้าง CLI tools เนื่องจาก compile เป็น binary เดียว ไม่ต้องมี runtime, และมี performance สูง ในบทนี้เราจะเรียนรู้วิธีสร้าง CLI tool ที่สมบูรณ์

## 1. Clap Crate (Derive API)

### พื้นฐาน clap

```toml
# Cargo.toml
[dependencies]
clap = { version = "4", features = ["derive", "env", "color"] }
colored = "2"
indicatif = "0.17"
dialoguer = "0.11"
toml = "0.8"
serde = { version = "1", features = ["derive"] }
anyhow = "1"
tokio = { version = "1", features = ["full"] }
sqlx = { version = "0.7", features = ["sqlite", "runtime-tokio-rustls"] }
comfy-table = "7"
```

```rust
use clap::{Parser, Subcommand, Args, ValueEnum};

#[derive(Parser, Debug)]
#[command(
    name = "dbcli",
    version = "1.0.0",
    about = "Database CLI Tool",
    long_about = "A powerful database management CLI tool built with Rust",
    author = "Your Name <email@example.com>"
)]
struct Cli {
    /// Increase output verbosity
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,
    
    /// Suppress all output
    #[arg(short, long)]
    quiet: bool,
    
    /// Config file path
    #[arg(short, long, value_name = "FILE", env = "DBCLI_CONFIG")]
    config: Option<String>,
    
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand, Debug)]
enum Commands {
    /// Database operations
    Db(DbArgs),
    
    /// User management
    User(UserArgs),
    
    /// Export data
    Export(ExportArgs),
    
    /// Import data
    Import(ImportArgs),
    
    /// Show statistics
    Stats {
        /// Filter by date range
        #[arg(long, value_name = "DATE")]
        from: Option<String>,
        
        #[arg(long, value_name = "DATE")]
        to: Option<String>,
        
        /// Output format
        #[arg(short, long, default_value = "table")]
        format: OutputFormat,
    },
}

#[derive(Args, Debug)]
struct DbArgs {
    #[command(subcommand)]
    action: DbAction,
}

#[derive(Subcommand, Debug)]
enum DbAction {
    /// Connect to database
    Connect {
        #[arg(short, long, env = "DATABASE_URL")]
        url: String,
    },
    
    /// Run migration
    Migrate {
        /// Migration direction
        #[arg(short, long, default_value = "up")]
        direction: MigrationDirection,
        
        /// Number of steps
        #[arg(short, long, default_value_t = 1)]
        steps: u32,
    },
    
    /// Execute query
    Query {
        /// SQL query to execute
        query: String,
        
        /// Query parameters
        #[arg(short, long)]
        param: Vec<String>,
    },
}

#[derive(Args, Debug)]
struct UserArgs {
    #[command(subcommand)]
    action: UserAction,
}

#[derive(Subcommand, Debug)]
enum UserAction {
    /// List all users
    List {
        #[arg(short, long, default_value_t = 10)]
        limit: u32,
        
        #[arg(short, long, default_value_t = 0)]
        offset: u32,
    },
    
    /// Create a new user
    Create {
        #[arg(short, long)]
        username: String,
        
        #[arg(short, long)]
        email: String,
        
        #[arg(short, long)]
        role: UserRole,
    },
    
    /// Delete a user
    Delete {
        /// User ID
        id: u64,
        
        /// Skip confirmation prompt
        #[arg(short, long)]
        force: bool,
    },
    
    /// Update user
    Update {
        id: u64,
        
        #[arg(long)]
        email: Option<String>,
        
        #[arg(long)]
        role: Option<UserRole>,
    },
}

#[derive(Args, Debug)]
struct ExportArgs {
    /// Output file
    #[arg(short, long, default_value = "export.csv")]
    output: String,
    
    /// Table to export
    #[arg(short, long)]
    table: String,
    
    /// Export format
    #[arg(short, long, default_value = "csv")]
    format: ExportFormat,
}

#[derive(Args, Debug)]
struct ImportArgs {
    /// Input file
    file: String,
    
    /// Target table
    #[arg(short, long)]
    table: String,
    
    /// Skip header row
    #[arg(long)]
    no_header: bool,
    
    /// Batch size
    #[arg(short, long, default_value_t = 100)]
    batch_size: u32,
}

#[derive(ValueEnum, Debug, Clone)]
enum OutputFormat {
    Table,
    Json,
    Csv,
    Yaml,
}

#[derive(ValueEnum, Debug, Clone)]
enum ExportFormat {
    Csv,
    Json,
    Sql,
}

#[derive(ValueEnum, Debug, Clone)]
enum UserRole {
    Admin,
    User,
    Moderator,
    ReadOnly,
}

#[derive(ValueEnum, Debug, Clone)]
enum MigrationDirection {
    Up,
    Down,
}
```

## 2. Subcommands Implementation

### Main dispatch logic

```rust
use anyhow::{Context, Result};
use std::path::PathBuf;

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    
    // Setup logging based on verbosity
    let log_level = match cli.verbose {
        0 => "warn",
        1 => "info",
        2 => "debug",
        _ => "trace",
    };
    
    env_logger::Builder::new()
        .parse_filters(log_level)
        .init();
    
    // Load config
    let config = load_config(cli.config.as_deref()).await?;
    
    // Dispatch command
    match cli.command {
        Commands::Db(args) => handle_db(args, &config).await,
        Commands::User(args) => handle_user(args, &config).await,
        Commands::Export(args) => handle_export(args, &config).await,
        Commands::Import(args) => handle_import(args, &config).await,
        Commands::Stats { from, to, format } => {
            handle_stats(from, to, format, &config).await
        }
    }
}

async fn handle_db(args: DbArgs, config: &Config) -> Result<()> {
    match args.action {
        DbAction::Connect { url } => {
            println!("Connecting to: {}", url);
            // Test connection
            let pool = connect_db(&url).await?;
            println!("{}", success_msg("Connected successfully!"));
            Ok(())
        }
        DbAction::Migrate { direction, steps } => {
            let direction_str = match direction {
                MigrationDirection::Up => "up",
                MigrationDirection::Down => "down",
            };
            println!("Running {} migration, {} step(s)...", direction_str, steps);
            // run migrations
            Ok(())
        }
        DbAction::Query { query, param } => {
            println!("Executing: {}", query);
            if !param.is_empty() {
                println!("Parameters: {:?}", param);
            }
            Ok(())
        }
    }
}

async fn handle_user(args: UserArgs, config: &Config) -> Result<()> {
    match args.action {
        UserAction::List { limit, offset } => {
            list_users(limit, offset, config).await
        }
        UserAction::Create { username, email, role } => {
            create_user(&username, &email, role, config).await
        }
        UserAction::Delete { id, force } => {
            delete_user(id, force, config).await
        }
        UserAction::Update { id, email, role } => {
            update_user(id, email, role, config).await
        }
    }
}
```

## 3. Output Formatting (Colors with colored)

### สีและ formatting

```rust
use colored::*;
use comfy_table::{Table, Cell, Attribute, Color as TableColor, ContentArrangement};

fn success_msg(msg: &str) -> ColoredString {
    format!("✓ {}", msg).green().bold()
}

fn error_msg(msg: &str) -> ColoredString {
    format!("✗ {}", msg).red().bold()
}

fn warning_msg(msg: &str) -> ColoredString {
    format!("⚠ {}", msg).yellow()
}

fn info_msg(msg: &str) -> ColoredString {
    format!("ℹ {}", msg).blue()
}

// Pretty table output
fn print_users_table(users: &[User]) {
    let mut table = Table::new();
    table
        .set_content_arrangement(ContentArrangement::Dynamic)
        .set_width(100)
        .set_header(vec![
            Cell::new("ID").add_attribute(Attribute::Bold),
            Cell::new("Username").add_attribute(Attribute::Bold),
            Cell::new("Email").add_attribute(Attribute::Bold),
            Cell::new("Role").add_attribute(Attribute::Bold),
            Cell::new("Created At").add_attribute(Attribute::Bold),
        ]);
    
    for user in users {
        let role_cell = match user.role.as_str() {
            "admin" => Cell::new(&user.role).fg(TableColor::Red),
            "moderator" => Cell::new(&user.role).fg(TableColor::Yellow),
            _ => Cell::new(&user.role).fg(TableColor::Green),
        };
        
        table.add_row(vec![
            Cell::new(user.id),
            Cell::new(&user.username),
            Cell::new(&user.email),
            role_cell,
            Cell::new(&user.created_at),
        ]);
    }
    
    println!("{table}");
    println!("\n{} {} users found", info_msg("→"), users.len());
}

// JSON output
fn print_users_json(users: &[User]) -> Result<()> {
    let json = serde_json::to_string_pretty(users)?;
    println!("{}", json);
    Ok(())
}

// CSV output
fn print_users_csv(users: &[User]) {
    println!("id,username,email,role,created_at");
    for user in users {
        println!("{},{},{},{},{}",
            user.id, user.username, user.email, user.role, user.created_at
        );
    }
}

// Syntax highlighting for SQL
fn highlight_sql(sql: &str) -> String {
    let keywords = ["SELECT", "FROM", "WHERE", "JOIN", "LEFT", "RIGHT", 
                   "INNER", "ON", "AND", "OR", "NOT", "IN", "LIKE",
                   "ORDER", "BY", "GROUP", "HAVING", "LIMIT", "OFFSET",
                   "INSERT", "INTO", "VALUES", "UPDATE", "SET", "DELETE",
                   "CREATE", "TABLE", "DROP", "ALTER", "ADD", "COLUMN"];
    
    let mut result = sql.to_string();
    for keyword in &keywords {
        result = result.replace(
            keyword,
            &keyword.blue().bold().to_string()
        );
    }
    result
}
```

## 4. Progress Bars (indicatif)

### Progress indicators

```rust
use indicatif::{ProgressBar, ProgressStyle, MultiProgress, ProgressDrawTarget};
use std::time::Duration;

fn create_progress_bar(len: u64, message: &str) -> ProgressBar {
    let pb = ProgressBar::new(len);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} {msg}")
            .unwrap()
            .progress_chars("#>-")
    );
    pb.set_message(message.to_string());
    pb
}

fn create_spinner(message: &str) -> ProgressBar {
    let pb = ProgressBar::new_spinner();
    pb.set_style(
        ProgressStyle::default_spinner()
            .tick_strings(&["⠋", "⠙", "⠹", "⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏"])
            .template("{spinner:.blue} {msg}")
            .unwrap()
    );
    pb.set_message(message.to_string());
    pb.enable_steady_tick(Duration::from_millis(80));
    pb
}

async fn import_with_progress(file: &str, table: &str, batch_size: u32) -> Result<()> {
    let spinner = create_spinner(&format!("Reading {}...", file));
    
    // Read file
    let data = tokio::fs::read_to_string(file).await
        .with_context(|| format!("Failed to read file: {}", file))?;
    
    let lines: Vec<&str> = data.lines().collect();
    spinner.finish_with_message(format!("Read {} lines", lines.len()));
    
    let pb = create_progress_bar(lines.len() as u64, "Importing...");
    
    let mut batch = Vec::new();
    let mut imported = 0u64;
    
    for line in &lines {
        batch.push(line);
        
        if batch.len() >= batch_size as usize {
            // Import batch
            tokio::time::sleep(Duration::from_millis(10)).await;
            imported += batch.len() as u64;
            pb.inc(batch.len() as u64);
            batch.clear();
        }
    }
    
    // Import remaining
    if !batch.is_empty() {
        imported += batch.len() as u64;
        pb.inc(batch.len() as u64);
    }
    
    pb.finish_with_message(format!("Imported {} records", imported));
    println!("{}", success_msg(&format!("Successfully imported {} records to '{}'", imported, table)));
    
    Ok(())
}

// Multi-progress bars
async fn parallel_export(tables: &[&str]) -> Result<()> {
    let multi = MultiProgress::new();
    let mut handles = vec![];
    
    for &table in tables {
        let pb = multi.add(ProgressBar::new(1000));
        pb.set_style(
            ProgressStyle::default_bar()
                .template(&format!("{{spinner:.green}} {} [{{bar:30}}] {{pos}}/{{len}}", table))
                .unwrap()
        );
        
        let table = table.to_string();
        handles.push(tokio::spawn(async move {
            for i in 0..1000 {
                tokio::time::sleep(Duration::from_millis(1)).await;
                pb.inc(1);
            }
            pb.finish_with_message("done");
            table
        }));
    }
    
    for handle in handles {
        let table = handle.await?;
        println!("{} Exported table: {}", success_msg("→"), table);
    }
    
    Ok(())
}
```

## 5. Config File Support

### TOML config file

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Default)]
struct Config {
    database: DatabaseConfig,
    output: OutputConfig,
    auth: AuthConfig,
}

#[derive(Debug, Serialize, Deserialize)]
struct DatabaseConfig {
    #[serde(default = "default_url")]
    url: String,
    
    #[serde(default = "default_pool_size")]
    pool_size: u32,
    
    #[serde(default = "default_timeout")]
    timeout_seconds: u64,
}

fn default_url() -> String {
    "sqlite://./data.db".to_string()
}

fn default_pool_size() -> u32 {
    10
}

fn default_timeout() -> u64 {
    30
}

impl Default for DatabaseConfig {
    fn default() -> Self {
        DatabaseConfig {
            url: default_url(),
            pool_size: default_pool_size(),
            timeout_seconds: default_timeout(),
        }
    }
}

#[derive(Debug, Serialize, Deserialize, Default)]
struct OutputConfig {
    #[serde(default)]
    format: String,
    
    #[serde(default)]
    color: bool,
    
    #[serde(default)]
    pager: bool,
}

#[derive(Debug, Serialize, Deserialize, Default)]
struct AuthConfig {
    #[serde(default)]
    token: Option<String>,
    
    #[serde(default)]
    username: Option<String>,
}

async fn load_config(path: Option<&str>) -> Result<Config> {
    // Try multiple locations
    let config_paths = vec![
        path.map(PathBuf::from),
        std::env::var("DBCLI_CONFIG").ok().map(PathBuf::from),
        Some(dirs::config_dir()
            .unwrap_or_default()
            .join("dbcli")
            .join("config.toml")),
        Some(PathBuf::from(".dbcli.toml")),
        Some(PathBuf::from("dbcli.toml")),
    ];
    
    for path in config_paths.into_iter().flatten() {
        if path.exists() {
            let content = tokio::fs::read_to_string(&path).await?;
            let config: Config = toml::from_str(&content)
                .with_context(|| format!("Failed to parse config: {}", path.display()))?;
            return Ok(config);
        }
    }
    
    // Return default config
    Ok(Config::default())
}

fn save_config(config: &Config, path: &str) -> Result<()> {
    let content = toml::to_string_pretty(config)?;
    std::fs::write(path, content)?;
    Ok(())
}

// Config subcommand
async fn handle_config_init() -> Result<()> {
    let config = Config {
        database: DatabaseConfig {
            url: "sqlite://./data.db".to_string(),
            pool_size: 10,
            timeout_seconds: 30,
        },
        output: OutputConfig {
            format: "table".to_string(),
            color: true,
            pager: false,
        },
        auth: AuthConfig::default(),
    };
    
    let config_dir = dirs::config_dir()
        .unwrap_or_default()
        .join("dbcli");
    
    tokio::fs::create_dir_all(&config_dir).await?;
    
    let config_path = config_dir.join("config.toml");
    let content = toml::to_string_pretty(&config)?;
    tokio::fs::write(&config_path, content).await?;
    
    println!("{} Config initialized at: {}", 
        success_msg("→"),
        config_path.display()
    );
    
    Ok(())
}
```

## 6. Practical: Database CLI Tool

### Complete CLI application

```rust
use clap::Parser;
use colored::*;
use indicatif::{ProgressBar, ProgressStyle};
use anyhow::{Context, Result};
use std::time::Duration;

// Mock data structures
#[derive(Debug, serde::Serialize)]
struct User {
    id: u64,
    username: String,
    email: String,
    role: String,
    created_at: String,
}

#[derive(Debug)]
struct Config {
    database_url: String,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            database_url: "sqlite://./data.db".to_string(),
        }
    }
}

async fn connect_db(url: &str) -> Result<String> {
    // Mock connection
    tokio::time::sleep(Duration::from_millis(100)).await;
    println!("{} Connected to: {}", "✓".green(), url);
    Ok("connection".to_string())
}

async fn list_users(limit: u32, offset: u32, config: &Config) -> Result<()> {
    let spinner = create_spinner("Fetching users...");
    tokio::time::sleep(Duration::from_millis(500)).await;
    spinner.finish_and_clear();
    
    // Mock users
    let users: Vec<User> = (offset..offset+limit.min(5))
        .map(|i| User {
            id: i as u64 + 1,
            username: format!("user_{}", i + 1),
            email: format!("user{}@example.com", i + 1),
            role: if i == 0 { "admin".to_string() } else { "user".to_string() },
            created_at: "2024-01-01".to_string(),
        })
        .collect();
    
    print_users_table(&users);
    Ok(())
}

async fn create_user(
    username: &str,
    email: &str,
    role: UserRole,
    config: &Config,
) -> Result<()> {
    let role_str = format!("{:?}", role).to_lowercase();
    
    // Validate
    if !email.contains('@') {
        anyhow::bail!("Invalid email address: {}", email);
    }
    
    let spinner = create_spinner(&format!("Creating user {}...", username));
    tokio::time::sleep(Duration::from_millis(300)).await;
    spinner.finish_and_clear();
    
    println!("{}", success_msg(&format!(
        "User '{}' created with role '{}'",
        username, role_str
    )));
    
    Ok(())
}

async fn delete_user(id: u64, force: bool, config: &Config) -> Result<()> {
    if !force {
        use dialoguer::Confirm;
        let confirmed = Confirm::new()
            .with_prompt(format!("Delete user #{}? This cannot be undone", id))
            .default(false)
            .interact()?;
        
        if !confirmed {
            println!("{}", warning_msg("Deletion cancelled"));
            return Ok(());
        }
    }
    
    let spinner = create_spinner(&format!("Deleting user #{}...", id));
    tokio::time::sleep(Duration::from_millis(200)).await;
    spinner.finish_and_clear();
    
    println!("{}", success_msg(&format!("User #{} deleted", id)));
    Ok(())
}

async fn update_user(
    id: u64,
    email: Option<String>,
    role: Option<UserRole>,
    config: &Config,
) -> Result<()> {
    if email.is_none() && role.is_none() {
        println!("{}", warning_msg("No updates specified"));
        return Ok(());
    }
    
    let spinner = create_spinner(&format!("Updating user #{}...", id));
    tokio::time::sleep(Duration::from_millis(200)).await;
    spinner.finish_and_clear();
    
    if let Some(email) = email {
        println!("  Email → {}", email.cyan());
    }
    if let Some(role) = role {
        println!("  Role → {}", format!("{:?}", role).cyan());
    }
    
    println!("{}", success_msg(&format!("User #{} updated", id)));
    Ok(())
}

async fn handle_export(args: ExportArgs, config: &Config) -> Result<()> {
    println!("Exporting table '{}' to '{}'...", 
        args.table.cyan(), 
        args.output.yellow()
    );
    
    let pb = create_progress_bar(100, "Exporting...");
    
    for i in 0..=100 {
        tokio::time::sleep(Duration::from_millis(20)).await;
        pb.set_position(i);
    }
    
    pb.finish_with_message("Export complete");
    println!("{}", success_msg(&format!("Data exported to '{}'", args.output)));
    
    Ok(())
}

async fn handle_import(args: ImportArgs, config: &Config) -> Result<()> {
    println!("Importing from '{}' to table '{}'...",
        args.file.cyan(),
        args.table.yellow()
    );
    
    import_with_progress(&args.file, &args.table, args.batch_size).await?;
    Ok(())
}

async fn handle_stats(
    from: Option<String>,
    to: Option<String>,
    format: OutputFormat,
    config: &Config,
) -> Result<()> {
    let spinner = create_spinner("Calculating statistics...");
    tokio::time::sleep(Duration::from_millis(500)).await;
    spinner.finish_and_clear();
    
    let stats = vec![
        ("Total Users", "1,234"),
        ("Active Users", "987"),
        ("New Users (Today)", "23"),
        ("Total Posts", "45,678"),
        ("Posts (Today)", "156"),
    ];
    
    match format {
        OutputFormat::Table => {
            let mut table = comfy_table::Table::new();
            table.set_header(vec!["Metric", "Value"]);
            for (k, v) in &stats {
                table.add_row(vec![k.to_string(), v.to_string()]);
            }
            println!("{table}");
        }
        OutputFormat::Json => {
            let json: serde_json::Value = stats.iter()
                .map(|(k, v)| (k.to_string(), serde_json::Value::String(v.to_string())))
                .collect::<serde_json::Map<_, _>>()
                .into();
            println!("{}", serde_json::to_string_pretty(&json)?);
        }
        OutputFormat::Csv => {
            println!("metric,value");
            for (k, v) in &stats {
                println!("{},{}", k, v);
            }
        }
        OutputFormat::Yaml => {
            for (k, v) in &stats {
                println!("{}: {}", k.to_lowercase().replace(' ', '_'), v);
            }
        }
    }
    
    Ok(())
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    let config = Config::default();
    
    let result = match cli.command {
        Commands::Db(args) => handle_db(args, &config).await,
        Commands::User(args) => handle_user(args, &config).await,
        Commands::Export(args) => handle_export(args, &config).await,
        Commands::Import(args) => handle_import(args, &config).await,
        Commands::Stats { from, to, format } => {
            handle_stats(from, to, format, &config).await
        }
    };
    
    if let Err(e) = result {
        eprintln!("{}: {}", "Error".red().bold(), e);
        std::process::exit(1);
    }
    
    Ok(())
}
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **Clap**: derive API สำหรับ argument parsing
- **Subcommands**: การจัดการ commands หลายระดับ
- **Environment Variables**: fallback จาก env vars
- **Config Files**: TOML config ด้วย multiple search paths
- **Output Formatting**: tables, JSON, CSV ด้วย comfy-table
- **Colors**: สีสันด้วย colored crate
- **Progress Bars**: indicators ด้วย indicatif
- **Database CLI**: ตัวอย่าง complete application

---

[← Part 095](../part_095/README.md) | [Part 097 →](../part_097/README.md)
