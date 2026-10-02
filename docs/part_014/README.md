# Part 014: File I/O และ Filesystem 📁

## 🎯 เป้าหมายของ Part นี้

- std::fs สำหรับการจัดการไฟล์
- std::path (Path และ PathBuf)
- BufReader และ BufWriter สำหรับไฟล์ขนาดใหญ่
- อ่านไฟล์ทีละบรรทัด
- CSV reading/writing
- JSON file I/O กับ serde_json
- Directory operations
- Environment variables ด้วย std::env

---

## 1. std::fs พื้นฐาน

```rust
use std::fs;
use std::io;

fn main() -> io::Result<()> {
    // 1. เขียนไฟล์ (overwrite ถ้ามีอยู่แล้ว)
    fs::write("hello.txt", "Hello, Rust File I/O!")?;
    println!("File written successfully");

    // 2. อ่านไฟล์เป็น String
    let content = fs::read_to_string("hello.txt")?;
    println!("Content: {}", content);

    // 3. อ่านไฟล์เป็น bytes
    let bytes = fs::read("hello.txt")?;
    println!("Bytes: {:?}", &bytes[..5]);

    // 4. เขียนข้อมูลหลายบรรทัด
    let multiline = "Line 1\nLine 2\nLine 3\nLine 4\nLine 5";
    fs::write("multiline.txt", multiline)?;

    // 5. Append ต่อท้ายไฟล์
    use std::io::Write;
    let mut file = fs::OpenOptions::new()
        .append(true)
        .open("hello.txt")?;
    writeln!(file, "\nAppended line!")?;

    // 6. Copy ไฟล์
    fs::copy("hello.txt", "hello_backup.txt")?;
    println!("File copied");

    // 7. ลบไฟล์
    fs::remove_file("hello_backup.txt")?;
    println!("Backup deleted");

    // 8. ตรวจสอบว่าไฟล์มีอยู่ไหม
    if fs::metadata("hello.txt").is_ok() {
        println!("hello.txt exists");
    }

    // 9. เปลี่ยนชื่อไฟล์
    fs::rename("multiline.txt", "renamed.txt")?;
    
    // 10. อ่าน metadata
    let meta = fs::metadata("hello.txt")?;
    println!("File size: {} bytes", meta.len());
    println!("Is file: {}", meta.is_file());
    println!("Is dir: {}", meta.is_dir());
    
    if let Ok(modified) = meta.modified() {
        println!("Modified: {:?}", modified);
    }

    // Cleanup
    fs::remove_file("hello.txt").ok();
    fs::remove_file("renamed.txt").ok();

    Ok(())
}
```

---

## 2. Path และ PathBuf

```rust
use std::path::{Path, PathBuf};
use std::env;

fn main() {
    // Path - immutable reference to path string
    let path = Path::new("/home/user/documents/file.txt");
    
    // PathBuf - owned, mutable path
    let mut buf = PathBuf::from("/home/user");

    // Path manipulation
    println!("Full path: {:?}", path);
    println!("Parent: {:?}", path.parent());
    println!("File name: {:?}", path.file_name());
    println!("File stem: {:?}", path.file_stem());   // file without extension
    println!("Extension: {:?}", path.extension());
    println!("Is absolute: {}", path.is_absolute());
    println!("Is relative: {}", Path::new("docs/file.txt").is_relative());

    // Building paths
    buf.push("documents");
    buf.push("projects");
    buf.push("rust");
    println!("Built path: {:?}", buf);  // /home/user/documents/projects/rust

    // Using join
    let home = PathBuf::from("/home/user");
    let config = home.join("config").join("app.toml");
    println!("Config path: {:?}", config);

    // Checking existence
    let current = env::current_dir().unwrap();
    println!("Current dir: {:?}", current);
    println!("Exists: {}", current.exists());

    // Path components
    let path = Path::new("/usr/local/bin/rustc");
    let components: Vec<_> = path.components().collect();
    println!("Components: {:?}", components);

    // Convert to string
    if let Some(s) = path.to_str() {
        println!("Path as str: {}", s);
    }
    
    // OS string
    let os_str = path.as_os_str();
    println!("OS string: {:?}", os_str);

    // Relative path
    let relative = PathBuf::from("src/main.rs");
    let absolute = current.join(&relative);
    println!("Absolute: {:?}", absolute);

    // Strip prefix
    let base = Path::new("/home/user");
    let full = Path::new("/home/user/documents/file.txt");
    if let Ok(rel) = full.strip_prefix(base) {
        println!("Relative to base: {:?}", rel);  // documents/file.txt
    }

    // File extension operations
    let mut p = PathBuf::from("document.txt");
    p.set_extension("pdf");
    println!("Changed extension: {:?}", p);  // document.pdf
    
    p.set_file_name("report.pdf");
    println!("Changed name: {:?}", p);  // report.pdf

    // Platform-specific separator
    let path_with_sep = PathBuf::from("parent")
        .join("child")
        .join("file.txt");
    println!("Platform path: {:?}", path_with_sep);
}
```

---

## 3. File struct - ควบคุมละเอียด

```rust
use std::fs::File;
use std::io::{self, Read, Write, Seek, SeekFrom};

fn main() -> io::Result<()> {
    // สร้างไฟล์ใหม่ (fails if exists)
    let mut file = File::create("new_file.txt")?;
    file.write_all(b"Hello, World!\n")?;
    file.write_all(b"Second line\n")?;
    file.flush()?;

    // เปิดไฟล์สำหรับอ่าน
    let mut file = File::open("new_file.txt")?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    println!("Read: {}", content);

    // OpenOptions - ควบคุมละเอียด
    let mut file = std::fs::OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .open("options_test.txt")?;
    
    file.write_all(b"Initial content\n")?;
    
    // Seek ไปยังตำแหน่งต่างๆ
    file.seek(SeekFrom::Start(0))?;  // ไปต้น
    let mut buf = String::new();
    file.read_to_string(&mut buf)?;
    println!("From start: {}", buf);
    
    // Seek จากท้าย
    file.seek(SeekFrom::End(-7))?;  // -7 จากท้าย
    let mut end_buf = vec![0u8; 7];
    file.read_exact(&mut end_buf)?;
    println!("Last 7 bytes: {:?}", String::from_utf8_lossy(&end_buf));

    // อ่าน bytes จำนวนเจาะจง
    file.seek(SeekFrom::Start(0))?;
    let mut chunk = vec![0u8; 7];  // อ่าน 7 bytes
    file.read_exact(&mut chunk)?;
    println!("First 7 bytes: {:?}", String::from_utf8_lossy(&chunk));

    // Current position
    let pos = file.seek(SeekFrom::Current(0))?;
    println!("Current position: {}", pos);

    // Cleanup
    std::fs::remove_file("new_file.txt").ok();
    std::fs::remove_file("options_test.txt").ok();

    Ok(())
}
```

---

## 4. BufReader และ BufWriter

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader, BufWriter, Write};
use std::path::Path;

fn main() -> io::Result<()> {
    // สร้างไฟล์ทดสอบ
    create_large_file("large.txt", 10000)?;
    
    // BufReader - อ่านไฟล์ขนาดใหญ่อย่างมีประสิทธิภาพ
    let file = File::open("large.txt")?;
    let reader = BufReader::new(file);
    
    let mut count = 0;
    let mut sum = 0u64;
    
    for line in reader.lines() {
        let line = line?;
        if let Ok(n) = line.trim().parse::<u64>() {
            sum += n;
            count += 1;
        }
    }
    
    println!("Lines: {}, Sum: {}", count, sum);

    // BufWriter - เขียนไฟล์ขนาดใหญ่อย่างมีประสิทธิภาพ
    let file = File::create("output.txt")?;
    let mut writer = BufWriter::new(file);
    
    for i in 0..1000 {
        writeln!(writer, "Line {}: {}", i, i * i)?;
    }
    
    // ต้อง flush เพื่อให้ข้อมูลใน buffer ถูกเขียนลงไฟล์
    writer.flush()?;
    println!("Written 1000 lines to output.txt");

    // BufReader กับ buffer size กำหนดเอง
    let file = File::open("output.txt")?;
    let reader = BufReader::with_capacity(64 * 1024, file);  // 64KB buffer
    
    let mut lines_read = 0;
    for line in reader.lines() {
        let _line = line?;
        lines_read += 1;
    }
    println!("Read {} lines", lines_read);

    // อ่านแบบ chunks
    let file = File::open("large.txt")?;
    let mut reader = BufReader::new(file);
    let mut total_bytes = 0;
    let mut buffer = vec![0u8; 4096];  // 4KB buffer
    
    loop {
        let bytes_read = reader.read(&mut buffer)?;
        if bytes_read == 0 { break; }
        total_bytes += bytes_read;
    }
    println!("Total bytes read: {}", total_bytes);

    // Cleanup
    std::fs::remove_file("large.txt").ok();
    std::fs::remove_file("output.txt").ok();

    Ok(())
}

fn create_large_file(path: &str, lines: usize) -> io::Result<()> {
    use std::io::Write;
    let file = File::create(path)?;
    let mut writer = BufWriter::new(file);
    for i in 0..lines {
        writeln!(writer, "{}", i)?;
    }
    writer.flush()
}
```

---

## 5. อ่านไฟล์ทีละบรรทัด

```rust
use std::fs::File;
use std::io::{self, BufRead, BufReader};
use std::path::Path;

// Helper function
fn read_lines<P: AsRef<Path>>(filename: P) -> io::Result<impl Iterator<Item = io::Result<String>>> {
    let file = File::open(filename)?;
    Ok(BufReader::new(file).lines())
}

fn process_log_file(path: &str) -> io::Result<LogStats> {
    let mut stats = LogStats::default();
    
    for line in read_lines(path)? {
        let line = line?;
        let trimmed = line.trim();
        
        if trimmed.is_empty() { continue; }
        
        stats.total_lines += 1;
        
        if trimmed.starts_with("ERROR") {
            stats.errors += 1;
            stats.error_messages.push(trimmed.to_string());
        } else if trimmed.starts_with("WARN") {
            stats.warnings += 1;
        } else if trimmed.starts_with("INFO") {
            stats.info_count += 1;
        }
    }
    
    Ok(stats)
}

#[derive(Debug, Default)]
struct LogStats {
    total_lines: usize,
    errors: usize,
    warnings: usize,
    info_count: usize,
    error_messages: Vec<String>,
}

fn main() -> io::Result<()> {
    // สร้าง test log file
    std::fs::write("app.log", 
        "INFO: Application started\n\
         INFO: Loading config\n\
         WARN: Config missing key 'timeout', using default\n\
         INFO: Connected to database\n\
         ERROR: Failed to process request: timeout\n\
         INFO: Retrying...\n\
         ERROR: Max retries exceeded\n\
         WARN: Falling back to cache\n\
         INFO: Shutdown complete\n"
    )?;
    
    let stats = process_log_file("app.log")?;
    println!("=== Log Analysis ===");
    println!("Total lines: {}", stats.total_lines);
    println!("Errors: {}", stats.errors);
    println!("Warnings: {}", stats.warnings);
    println!("Info: {}", stats.info_count);
    println!("\nError details:");
    for err in &stats.error_messages {
        println!("  - {}", err);
    }
    
    // อ่านแบบ enumerate
    println!("\n=== Line Numbers ===");
    for (i, line) in read_lines("app.log")?.enumerate() {
        let line = line?;
        if line.starts_with("ERROR") {
            println!("Line {}: {}", i + 1, line);
        }
    }
    
    // อ่าน N บรรทัดแรก
    println!("\n=== First 3 lines ===");
    for line in read_lines("app.log")?.take(3) {
        println!("{}", line?);
    }

    // อ่านและ filter
    let errors: Vec<String> = read_lines("app.log")?
        .filter_map(|line| {
            let line = line.ok()?;
            if line.starts_with("ERROR") { Some(line) } else { None }
        })
        .collect();
    
    println!("\nAll errors: {:?}", errors);
    
    std::fs::remove_file("app.log").ok();
    Ok(())
}
```

---

## 6. CSV Reading/Writing

```toml
# Cargo.toml
[dependencies]
csv = "1.3"
serde = { version = "1.0", features = ["derive"] }
```

```rust
use std::error::Error;
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
struct Student {
    id: u32,
    name: String,
    grade: String,
    score: f64,
    passed: bool,
}

impl Student {
    fn new(id: u32, name: &str, grade: &str, score: f64) -> Self {
        Student {
            id,
            name: name.to_string(),
            grade: grade.to_string(),
            score,
            passed: score >= 60.0,
        }
    }
}

fn write_students_csv(students: &[Student], path: &str) -> Result<(), Box<dyn Error>> {
    let mut writer = csv::Writer::from_path(path)?;
    
    for student in students {
        writer.serialize(student)?;
    }
    
    writer.flush()?;
    println!("CSV written: {}", path);
    Ok(())
}

fn read_students_csv(path: &str) -> Result<Vec<Student>, Box<dyn Error>> {
    let mut reader = csv::Reader::from_path(path)?;
    let mut students = Vec::new();
    
    for result in reader.deserialize() {
        let student: Student = result?;
        students.push(student);
    }
    
    Ok(students)
}

fn read_csv_manual(path: &str) -> Result<(), Box<dyn Error>> {
    let mut reader = csv::ReaderBuilder::new()
        .has_headers(true)
        .delimiter(b',')
        .from_path(path)?;
    
    // Read headers
    let headers = reader.headers()?.clone();
    println!("Headers: {:?}", headers);
    
    // Read records
    for result in reader.records() {
        let record = result?;
        for (header, field) in headers.iter().zip(record.iter()) {
            print!("{}: {} | ", header, field);
        }
        println!();
    }
    
    Ok(())
}

fn write_csv_manual(path: &str) -> Result<(), Box<dyn Error>> {
    let mut writer = csv::WriterBuilder::new()
        .delimiter(b',')
        .quote_style(csv::QuoteStyle::Necessary)
        .from_path(path)?;
    
    // Write header
    writer.write_record(&["id", "name", "score", "comment"])?;
    
    // Write records
    let data = vec![
        ("1", "Alice", "95.5", "Excellent"),
        ("2", "Bob", "78.0", "Good"),
        ("3", "Charlie", "62.5", "Needs improvement"),
    ];
    
    for row in data {
        writer.write_record(&[row.0, row.1, row.2, row.3])?;
    }
    
    writer.flush()?;
    Ok(())
}

fn analyze_students(students: &[Student]) {
    let total = students.len();
    let passed = students.iter().filter(|s| s.passed).count();
    let avg = students.iter().map(|s| s.score).sum::<f64>() / total as f64;
    let max = students.iter().map(|s| s.score).fold(f64::NEG_INFINITY, f64::max);
    let min = students.iter().map(|s| s.score).fold(f64::INFINITY, f64::min);
    
    println!("=== Student Analysis ===");
    println!("Total: {}", total);
    println!("Passed: {}/{} ({:.1}%)", passed, total, passed as f64 / total as f64 * 100.0);
    println!("Average: {:.1}", avg);
    println!("Max: {:.1}", max);
    println!("Min: {:.1}", min);
    
    println!("\nTop 3 students:");
    let mut sorted = students.to_vec();
    sorted.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());
    for (i, s) in sorted.iter().take(3).enumerate() {
        println!("  {}. {} - {:.1}", i + 1, s.name, s.score);
    }
}

fn main() -> Result<(), Box<dyn Error>> {
    let students = vec![
        Student::new(1, "Alice", "A", 95.0),
        Student::new(2, "Bob", "B", 82.0),
        Student::new(3, "Charlie", "C", 70.0),
        Student::new(4, "Diana", "A", 98.5),
        Student::new(5, "Eve", "D", 55.0),
        Student::new(6, "Frank", "B", 78.0),
        Student::new(7, "Grace", "F", 45.0),
        Student::new(8, "Henry", "C", 65.0),
    ];
    
    // Write CSV
    write_students_csv(&students, "students.csv")?;
    
    // Read CSV back
    let loaded = read_students_csv("students.csv")?;
    analyze_students(&loaded);
    
    // Manual reading
    println!("\n=== Manual CSV Reading ===");
    read_csv_manual("students.csv")?;
    
    // Write manual CSV
    write_csv_manual("manual.csv")?;
    
    // Cleanup
    std::fs::remove_file("students.csv").ok();
    std::fs::remove_file("manual.csv").ok();
    
    Ok(())
}
```

---

## 7. JSON File I/O กับ serde_json

```toml
# Cargo.toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

```rust
use std::fs;
use std::io;
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};
use std::collections::HashMap;

#[derive(Debug, Serialize, Deserialize, Clone)]
struct AppConfig {
    version: String,
    server: ServerConfig,
    database: DatabaseConfig,
    features: Vec<String>,
    settings: HashMap<String, serde_json::Value>,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct ServerConfig {
    host: String,
    port: u16,
    max_connections: u32,
    timeout_seconds: u64,
}

#[derive(Debug, Serialize, Deserialize, Clone)]
struct DatabaseConfig {
    url: String,
    pool_size: u32,
    #[serde(skip_serializing_if = "Option::is_none")]
    password: Option<String>,
}

fn load_config(path: &str) -> Result<AppConfig, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let config: AppConfig = serde_json::from_str(&content)?;
    Ok(config)
}

fn save_config(config: &AppConfig, path: &str) -> Result<(), Box<dyn std::error::Error>> {
    let json = serde_json::to_string_pretty(config)?;
    fs::write(path, json)?;
    Ok(())
}

fn load_json_value(path: &str) -> Result<Value, Box<dyn std::error::Error>> {
    let content = fs::read_to_string(path)?;
    let value: Value = serde_json::from_str(&content)?;
    Ok(value)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // สร้าง config
    let mut settings = HashMap::new();
    settings.insert("debug_mode".to_string(), json!(false));
    settings.insert("log_level".to_string(), json!("info"));
    settings.insert("cache_ttl".to_string(), json!(3600));
    
    let config = AppConfig {
        version: "1.0.0".to_string(),
        server: ServerConfig {
            host: "0.0.0.0".to_string(),
            port: 8080,
            max_connections: 1000,
            timeout_seconds: 30,
        },
        database: DatabaseConfig {
            url: "postgres://localhost/myapp".to_string(),
            pool_size: 10,
            password: None,
        },
        features: vec![
            "authentication".to_string(),
            "rate_limiting".to_string(),
            "caching".to_string(),
        ],
        settings,
    };
    
    // Save to file
    save_config(&config, "config.json")?;
    println!("Config saved to config.json");
    
    // Load from file
    let loaded = load_config("config.json")?;
    println!("Loaded config: version={}, port={}", 
        loaded.version, loaded.server.port);
    println!("Features: {:?}", loaded.features);
    
    // JSON manipulation with Value
    let json_str = r#"
    {
        "users": [
            {"id": 1, "name": "Alice", "active": true},
            {"id": 2, "name": "Bob", "active": false},
            {"id": 3, "name": "Charlie", "active": true}
        ],
        "total": 3,
        "page": 1
    }
    "#;
    
    let value: Value = serde_json::from_str(json_str)?;
    
    // Navigate JSON
    println!("\nTotal: {}", value["total"]);
    println!("Page: {}", value["page"]);
    
    if let Some(users) = value["users"].as_array() {
        println!("\nActive users:");
        for user in users {
            if user["active"].as_bool().unwrap_or(false) {
                println!("  - {} (id: {})", user["name"], user["id"]);
            }
        }
    }
    
    // Modify JSON
    let mut data: Value = serde_json::from_str(json_str)?;
    data["total"] = json!(10);
    data["page"] = json!(2);
    
    if let Some(users) = data["users"].as_array_mut() {
        users.push(json!({"id": 4, "name": "Diana", "active": true}));
    }
    
    fs::write("users.json", serde_json::to_string_pretty(&data)?)?;
    println!("\nModified JSON saved");
    
    // Reading large JSON file line by line (NDJSON format)
    let ndjson = r#"{"id":1,"event":"login","user":"alice"}
{"id":2,"event":"purchase","user":"bob","amount":99.99}
{"id":3,"event":"logout","user":"alice"}
{"id":4,"event":"login","user":"charlie"}"#;
    
    fs::write("events.ndjson", ndjson)?;
    
    println!("\nNDJSON events:");
    for line in fs::read_to_string("events.ndjson")?.lines() {
        if line.trim().is_empty() { continue; }
        let event: Value = serde_json::from_str(line)?;
        println!("  Event: {} by {}", event["event"], event["user"]);
    }
    
    // JSON with custom serialization
    #[derive(Serialize, Deserialize, Debug)]
    struct Timestamp {
        #[serde(with = "timestamp_format")]
        created_at: std::time::SystemTime,
    }
    
    // Cleanup
    fs::remove_file("config.json").ok();
    fs::remove_file("users.json").ok();
    fs::remove_file("events.ndjson").ok();
    
    Ok(())
}

// Custom serialization module
mod timestamp_format {
    use serde::{self, Deserialize, Deserializer, Serializer};
    use std::time::{SystemTime, UNIX_EPOCH};
    
    pub fn serialize<S>(time: &SystemTime, serializer: S) -> Result<S::Ok, S::Error>
    where S: Serializer {
        let secs = time.duration_since(UNIX_EPOCH)
            .map_err(serde::ser::Error::custom)?
            .as_secs();
        serializer.serialize_u64(secs)
    }
    
    pub fn deserialize<'de, D>(deserializer: D) -> Result<SystemTime, D::Error>
    where D: Deserializer<'de> {
        let secs = u64::deserialize(deserializer)?;
        Ok(UNIX_EPOCH + std::time::Duration::from_secs(secs))
    }
}
```

---

## 8. Directory Operations

```rust
use std::fs;
use std::path::{Path, PathBuf};
use std::io;

fn main() -> io::Result<()> {
    // สร้าง directory
    fs::create_dir("mydir")?;
    println!("Created mydir");
    
    // สร้าง nested directories
    fs::create_dir_all("project/src/utils")?;
    fs::create_dir_all("project/tests")?;
    fs::create_dir_all("project/docs")?;
    println!("Created project structure");
    
    // สร้างไฟล์ใน directories
    fs::write("project/src/main.rs", "fn main() {}")?;
    fs::write("project/src/utils/helper.rs", "pub fn help() {}")?;
    fs::write("project/tests/test_main.rs", "#[test] fn it_works() {}")?;
    fs::write("project/Cargo.toml", "[package]\nname = \"project\"")?;
    
    // อ่านรายการไฟล์ใน directory
    println!("\nContents of project/:");
    for entry in fs::read_dir("project")? {
        let entry = entry?;
        let path = entry.path();
        let name = entry.file_name();
        let is_dir = entry.file_type()?.is_dir();
        println!("  {} {}", if is_dir { "📁" } else { "📄" }, name.to_string_lossy());
    }
    
    // Walk directory recursively
    println!("\nAll files in project/:");
    walk_dir(Path::new("project"), 0)?;
    
    // Find files by extension
    println!("\nRust files:");
    let rust_files = find_files_by_ext(Path::new("project"), "rs")?;
    for f in &rust_files {
        println!("  {:?}", f);
    }
    
    // Get directory info
    let dir_size = get_dir_size(Path::new("project"))?;
    println!("\nTotal size: {} bytes", dir_size);
    
    // Copy directory (recursive)
    copy_dir(Path::new("project"), Path::new("project_backup"))?;
    println!("Directory copied to project_backup");
    
    // Remove directories
    fs::remove_dir("mydir")?;  // fails if not empty
    fs::remove_dir_all("project")?;
    fs::remove_dir_all("project_backup")?;
    println!("Directories removed");
    
    Ok(())
}

fn walk_dir(dir: &Path, depth: usize) -> io::Result<()> {
    let indent = "  ".repeat(depth);
    
    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path = entry.path();
        let name = entry.file_name();
        
        if path.is_dir() {
            println!("{}📁 {}/", indent, name.to_string_lossy());
            walk_dir(&path, depth + 1)?;
        } else {
            println!("{}📄 {}", indent, name.to_string_lossy());
        }
    }
    
    Ok(())
}

fn find_files_by_ext(dir: &Path, ext: &str) -> io::Result<Vec<PathBuf>> {
    let mut results = Vec::new();
    
    if dir.is_dir() {
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            
            if path.is_dir() {
                let sub = find_files_by_ext(&path, ext)?;
                results.extend(sub);
            } else if path.extension().and_then(|e| e.to_str()) == Some(ext) {
                results.push(path);
            }
        }
    }
    
    Ok(results)
}

fn get_dir_size(dir: &Path) -> io::Result<u64> {
    let mut size = 0u64;
    
    if dir.is_dir() {
        for entry in fs::read_dir(dir)? {
            let entry = entry?;
            let path = entry.path();
            
            if path.is_dir() {
                size += get_dir_size(&path)?;
            } else {
                size += entry.metadata()?.len();
            }
        }
    }
    
    Ok(size)
}

fn copy_dir(src: &Path, dst: &Path) -> io::Result<()> {
    fs::create_dir_all(dst)?;
    
    for entry in fs::read_dir(src)? {
        let entry = entry?;
        let src_path = entry.path();
        let dst_path = dst.join(entry.file_name());
        
        if src_path.is_dir() {
            copy_dir(&src_path, &dst_path)?;
        } else {
            fs::copy(&src_path, &dst_path)?;
        }
    }
    
    Ok(())
}
```

---

## 9. Environment Variables

```rust
use std::env;
use std::collections::HashMap;

fn main() {
    // อ่าน environment variable
    let path = env::var("PATH").unwrap_or_default();
    println!("PATH (first 100 chars): {}...", &path[..path.len().min(100)]);
    
    // อ่านด้วย default value
    let log_level = env::var("LOG_LEVEL").unwrap_or_else(|_| "info".to_string());
    println!("Log level: {}", log_level);
    
    // ตรวจสอบว่ามี variable ไหม
    match env::var("DATABASE_URL") {
        Ok(url) => println!("DB URL: {}", url),
        Err(env::VarError::NotPresent) => println!("DATABASE_URL not set"),
        Err(env::VarError::NotUnicode(s)) => println!("Invalid unicode: {:?}", s),
    }
    
    // ตั้งค่า environment variable (เฉพาะ process ปัจจุบัน)
    env::set_var("MY_APP_VERSION", "1.0.0");
    println!("App version: {}", env::var("MY_APP_VERSION").unwrap());
    
    // ลบ environment variable
    env::remove_var("MY_APP_VERSION");
    println!("After remove: {:?}", env::var("MY_APP_VERSION"));
    
    // อ่าน environment variables ทั้งหมด
    let env_vars: HashMap<String, String> = env::vars().collect();
    println!("Total env vars: {}", env_vars.len());
    
    // Current executable path
    if let Ok(exe) = env::current_exe() {
        println!("Executable: {:?}", exe);
    }
    
    // Current directory
    let current_dir = env::current_dir().unwrap();
    println!("Current dir: {:?}", current_dir);
    
    // Arguments ที่ส่งมากับโปรแกรม
    let args: Vec<String> = env::args().collect();
    println!("Args: {:?}", args);
    
    // การใช้ env vars สำหรับ config
    let config = AppConfig::from_env();
    println!("\nApp Config: {:?}", config);
}

#[derive(Debug)]
struct AppConfig {
    host: String,
    port: u16,
    debug: bool,
    workers: usize,
}

impl AppConfig {
    fn from_env() -> Self {
        AppConfig {
            host: env::var("APP_HOST").unwrap_or_else(|_| "127.0.0.1".to_string()),
            port: env::var("APP_PORT")
                .ok()
                .and_then(|p| p.parse().ok())
                .unwrap_or(8080),
            debug: env::var("APP_DEBUG")
                .map(|v| v.to_lowercase() == "true" || v == "1")
                .unwrap_or(false),
            workers: env::var("APP_WORKERS")
                .ok()
                .and_then(|w| w.parse().ok())
                .unwrap_or_else(|| num_cpus()),
        }
    }
}

fn num_cpus() -> usize {
    // Simple fallback
    std::thread::available_parallelism()
        .map(|n| n.get())
        .unwrap_or(4)
}
```

---

## 10. dotenv - โหลด .env file

```toml
[dependencies]
dotenvy = "0.15"
```

```rust
// .env file ตัวอย่าง:
// DATABASE_URL=postgres://localhost/myapp
// SECRET_KEY=super_secret_key_123
// DEBUG=true
// MAX_CONNECTIONS=50

fn load_env_config() {
    // โหลด .env file (ถ้ามี)
    dotenvy::dotenv().ok();  // .ok() เพื่อไม่ error ถ้าไม่มีไฟล์
    
    // หรือโหลดจาก path เจาะจง
    // dotenvy::from_path(".env.production").ok();
    
    let db_url = std::env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    
    let debug = std::env::var("DEBUG")
        .unwrap_or_else(|_| "false".to_string())
        .parse::<bool>()
        .unwrap_or(false);
    
    println!("Database: {}", db_url);
    println!("Debug mode: {}", debug);
}
```

---

## 11. Practical: File Processing Tool

```rust
use std::fs;
use std::io::{self, BufRead, BufReader, BufWriter, Write};
use std::path::{Path, PathBuf};
use std::collections::HashMap;

struct FileProcessor {
    input_path: PathBuf,
    output_path: PathBuf,
}

impl FileProcessor {
    fn new(input: &str, output: &str) -> Self {
        FileProcessor {
            input_path: PathBuf::from(input),
            output_path: PathBuf::from(output),
        }
    }
    
    fn count_words(&self) -> io::Result<HashMap<String, usize>> {
        let file = fs::File::open(&self.input_path)?;
        let reader = BufReader::new(file);
        let mut word_count = HashMap::new();
        
        for line in reader.lines() {
            let line = line?;
            for word in line.split_whitespace() {
                let clean: String = word.chars()
                    .filter(|c| c.is_alphabetic())
                    .map(|c| c.to_lowercase().next().unwrap())
                    .collect();
                
                if !clean.is_empty() {
                    *word_count.entry(clean).or_insert(0) += 1;
                }
            }
        }
        
        Ok(word_count)
    }
    
    fn line_stats(&self) -> io::Result<LineStats> {
        let file = fs::File::open(&self.input_path)?;
        let reader = BufReader::new(file);
        let mut stats = LineStats::default();
        
        for line in reader.lines() {
            let line = line?;
            stats.total_lines += 1;
            
            if line.trim().is_empty() {
                stats.empty_lines += 1;
            } else {
                let char_count = line.chars().count();
                stats.total_chars += char_count;
                stats.max_line_length = stats.max_line_length.max(char_count);
                stats.min_line_length = stats.min_line_length.min(char_count);
            }
        }
        
        if stats.total_lines > stats.empty_lines {
            let non_empty = stats.total_lines - stats.empty_lines;
            stats.avg_line_length = stats.total_chars as f64 / non_empty as f64;
        }
        
        Ok(stats)
    }
    
    fn transform<F>(&self, transform_fn: F) -> io::Result<usize>
    where F: Fn(&str) -> String
    {
        let in_file = fs::File::open(&self.input_path)?;
        let reader = BufReader::new(in_file);
        
        let out_file = fs::File::create(&self.output_path)?;
        let mut writer = BufWriter::new(out_file);
        
        let mut count = 0;
        for line in reader.lines() {
            let line = line?;
            let transformed = transform_fn(&line);
            writeln!(writer, "{}", transformed)?;
            count += 1;
        }
        
        writer.flush()?;
        Ok(count)
    }
    
    fn search(&self, pattern: &str) -> io::Result<Vec<(usize, String)>> {
        let file = fs::File::open(&self.input_path)?;
        let reader = BufReader::new(file);
        let mut matches = Vec::new();
        
        for (i, line) in reader.lines().enumerate() {
            let line = line?;
            if line.contains(pattern) {
                matches.push((i + 1, line));
            }
        }
        
        Ok(matches)
    }
    
    fn split_by_lines(&self, lines_per_file: usize) -> io::Result<Vec<PathBuf>> {
        let file = fs::File::open(&self.input_path)?;
        let reader = BufReader::new(file);
        
        let stem = self.output_path.file_stem()
            .and_then(|s| s.to_str())
            .unwrap_or("part");
        let ext = self.output_path.extension()
            .and_then(|e| e.to_str())
            .unwrap_or("txt");
        let parent = self.output_path.parent()
            .unwrap_or(Path::new("."));
        
        let mut output_files = Vec::new();
        let mut current_file: Option<BufWriter<fs::File>> = None;
        let mut line_count = 0;
        let mut file_num = 0;
        
        for line in reader.lines() {
            let line = line?;
            
            if current_file.is_none() || line_count >= lines_per_file {
                file_num += 1;
                let path = parent.join(format!("{}_{:04}.{}", stem, file_num, ext));
                let f = fs::File::create(&path)?;
                output_files.push(path);
                current_file = Some(BufWriter::new(f));
                line_count = 0;
            }
            
            if let Some(ref mut writer) = current_file {
                writeln!(writer, "{}", line)?;
                line_count += 1;
            }
        }
        
        if let Some(mut writer) = current_file {
            writer.flush()?;
        }
        
        Ok(output_files)
    }
}

#[derive(Debug, Default)]
struct LineStats {
    total_lines: usize,
    empty_lines: usize,
    total_chars: usize,
    max_line_length: usize,
    min_line_length: usize,
    avg_line_length: f64,
}

fn main() -> io::Result<()> {
    // สร้าง test file
    let content = "The quick brown fox jumps over the lazy dog.
This is a test file for our file processor.
It contains multiple lines of text.

Some lines are longer than others, demonstrating the line length statistics.
Short line.
Another test line with some words.

The processor can count words, analyze lines, and transform content.";
    
    fs::write("input.txt", content)?;
    
    let processor = FileProcessor::new("input.txt", "output.txt");
    
    // 1. Count words
    let word_count = processor.count_words()?;
    let mut words: Vec<_> = word_count.iter().collect();
    words.sort_by(|a, b| b.1.cmp(a.1));
    
    println!("=== Top 10 Words ===");
    for (word, count) in words.iter().take(10) {
        println!("  {:15} : {}", word, count);
    }
    println!("  Total unique words: {}", word_count.len());
    
    // 2. Line stats
    let stats = processor.line_stats()?;
    println!("\n=== Line Statistics ===");
    println!("  Total lines: {}", stats.total_lines);
    println!("  Empty lines: {}", stats.empty_lines);
    println!("  Total chars: {}", stats.total_chars);
    println!("  Max length:  {}", stats.max_line_length);
    println!("  Min length:  {}", stats.min_line_length);
    println!("  Avg length:  {:.1}", stats.avg_line_length);
    
    // 3. Transform - uppercase
    let lines = processor.transform(|line| line.to_uppercase())?;
    println!("\n=== Transform (uppercase) ===");
    println!("  Transformed {} lines", lines);
    println!("  First line: {}", fs::read_to_string("output.txt")?.lines().next().unwrap_or(""));
    
    // 4. Search
    let matches = processor.search("test")?;
    println!("\n=== Search 'test' ===");
    for (line_num, line) in &matches {
        println!("  Line {}: {}", line_num, line);
    }
    
    // 5. Split file
    let parts = processor.split_by_lines(3)?;
    println!("\n=== Split into parts ===");
    println!("  Created {} files:", parts.len());
    for p in &parts {
        let content = fs::read_to_string(p)?;
        println!("  {:?}: {} lines", p.file_name().unwrap(), content.lines().count());
    }
    
    // Cleanup
    fs::remove_file("input.txt").ok();
    fs::remove_file("output.txt").ok();
    for p in parts {
        fs::remove_file(p).ok();
    }
    
    println!("\nDone!");
    Ok(())
}
```

---

## 12. สรุป

| Function/Type | Description | ตัวอย่าง |
|---------------|-------------|---------|
| `fs::read_to_string` | อ่านทั้งไฟล์เป็น String | `fs::read_to_string("file.txt")?` |
| `fs::write` | เขียนข้อมูลลงไฟล์ | `fs::write("f.txt", "data")?` |
| `fs::copy` | copy ไฟล์ | `fs::copy("src", "dst")?` |
| `fs::remove_file` | ลบไฟล์ | `fs::remove_file("f.txt")?` |
| `fs::create_dir_all` | สร้าง nested dirs | `fs::create_dir_all("a/b/c")?` |
| `fs::remove_dir_all` | ลบ directory | `fs::remove_dir_all("dir")?` |
| `fs::read_dir` | list directory | `for entry in fs::read_dir(".")?` |
| `Path::new` | immutable path | `Path::new("/usr/bin")` |
| `PathBuf::from` | owned path | `PathBuf::from("/home/user")` |
| `BufReader` | buffered read | `BufReader::new(file).lines()` |
| `BufWriter` | buffered write | `BufWriter::new(file)` |
| `env::var` | อ่าน env var | `env::var("HOME")?` |

### Tips สำคัญ

1. **ใช้ BufReader/BufWriter** สำหรับไฟล์ขนาดใหญ่
2. **? operator** สำหรับ error propagation ใน I/O operations
3. **PathBuf** เมื่อต้องการ mutable path, **Path** เมื่อ reference
4. **สร้าง parent dirs** ด้วย `create_dir_all` เสมอ
5. **flush() BufWriter** เสมอก่อน drop

---

*[← Part 013: Testing ใน Rust](../part_013/README.md) | [Part 015: Concurrency กับ Threads →](../part_015/README.md)*
