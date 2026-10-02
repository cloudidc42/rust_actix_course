# Part 019: String Handling Deep Dive

## บทนำ

String ใน Rust มีความพิเศษกว่าภาษาอื่นๆ เพราะ Rust แยกแยะอย่างชัดเจนระหว่าง owned string (`String`) กับ string slice (`&str`) ในบทนี้เราจะเรียนรู้การจัดการ strings อย่างละเอียดรวมถึง Unicode, Regular Expressions และ text processing utilities

---

## 1. String vs &str

### ความแตกต่างพื้นฐาน

```rust
fn main() {
    // &str - string slice
    // - reference ไปยัง UTF-8 encoded bytes
    // - ขนาดคงที่ตอน compile time หรือ runtime slice
    // - ไม่ต้องการ heap allocation (สำหรับ literal)
    let s1: &str = "hello"; // string literal อยู่ใน binary
    let s2: &'static str = "world"; // static lifetime
    
    // String - heap-allocated string
    // - owned, growable, heap-allocated
    // - มี length และ capacity
    // - drop เมื่อออกจาก scope
    let s3: String = String::from("hello");
    let s4: String = "hello".to_string();
    let s5: String = "hello".to_owned();
    let s6 = String::with_capacity(50); // pre-allocated
    
    // แปลงระหว่างกัน
    let slice: &str = &s3;        // String -> &str (borrow)
    let owned: String = s1.to_string(); // &str -> String
    let owned2: String = s1.to_owned(); // &str -> String
    let owned3: String = String::from(s1); // &str -> String
    
    // ขนาดใน memory
    println!("&str size: {} bytes (pointer + length)", std::mem::size_of::<&str>());
    println!("String size: {} bytes (pointer + length + capacity)", std::mem::size_of::<String>());
    
    // String internals
    let s = String::from("hello");
    println!("len: {}", s.len());          // จำนวน bytes
    println!("capacity: {}", s.capacity()); // bytes ที่ allocate
    println!("is_empty: {}", s.is_empty());
}
```

### Memory Layout

```rust
fn demonstrate_memory() {
    // Stack: pointer (8), length (8) = 16 bytes
    let slice: &str = "Hello, World!";
    
    // Heap: actual bytes
    // Stack: pointer (8), length (8), capacity (8) = 24 bytes  
    let owned = String::from("Hello, World!");
    
    println!("Slice: ptr={:p}, len={}", slice.as_ptr(), slice.len());
    println!("Owned: ptr={:p}, len={}, cap={}", 
        owned.as_ptr(), owned.len(), owned.capacity());
    
    // String เก็บ UTF-8 bytes
    let thai = "สวัสดี";
    println!("\nThai string: {}", thai);
    println!("Byte length: {}", thai.len()); // bytes (ไม่ใช่ chars)
    println!("Char count: {}", thai.chars().count()); // characters
    
    // การเข้าถึง bytes
    for (i, byte) in thai.bytes().enumerate().take(6) {
        println!("Byte {}: 0x{:02X}", i, byte);
    }
}

fn main() {
    demonstrate_memory();
}
```

---

## 2. String Methods

### contains, starts_with, ends_with

```rust
fn string_search_methods() {
    let text = "The quick brown fox jumps over the lazy dog";
    
    // contains - ตรวจสอบว่ามี substring
    println!("Contains 'fox': {}", text.contains("fox"));
    println!("Contains 'cat': {}", text.contains("cat"));
    
    // starts_with
    println!("Starts with 'The': {}", text.starts_with("The"));
    println!("Starts with 'A': {}", text.starts_with('A'));
    
    // ends_with
    println!("Ends with 'dog': {}", text.ends_with("dog"));
    println!("Ends with 'g': {}", text.ends_with('g'));
    
    // Pattern matching
    let patterns = ["quick", "lazy", "brave"];
    for pat in &patterns {
        println!("Contains '{}': {}", pat, text.contains(*pat));
    }
    
    // find และ rfind - หา index ของ substring
    if let Some(pos) = text.find("fox") {
        println!("'fox' at index: {}", pos);
        println!("Substring: {}", &text[pos..pos+3]);
    }
    
    // rfind - ค้นหาจากขวา
    let repeated = "hello world hello";
    println!("Last 'hello' at: {:?}", repeated.rfind("hello"));
}
```

### split และ split_whitespace

```rust
fn string_split_methods() {
    // split - แบ่ง string ตาม separator
    let csv = "name,age,email,phone";
    let fields: Vec<&str> = csv.split(',').collect();
    println!("Fields: {:?}", fields);
    
    // splitn - จำกัดจำนวนครั้งที่แบ่ง
    let limited: Vec<&str> = csv.splitn(3, ',').collect();
    println!("Limited split: {:?}", limited);
    
    // split_whitespace - แบ่งด้วย whitespace ใดก็ได้
    let sentence = "  hello   world   rust  ";
    let words: Vec<&str> = sentence.split_whitespace().collect();
    println!("Words: {:?}", words);
    
    // lines - แบ่งด้วย newline
    let multiline = "line1\nline2\r\nline3\nline4";
    for (i, line) in multiline.lines().enumerate() {
        println!("Line {}: {}", i + 1, line);
    }
    
    // split_once - แบ่งครั้งเดียว
    let key_value = "name=John Doe";
    if let Some((key, value)) = key_value.split_once('=') {
        println!("Key: {}, Value: {}", key, value);
    }
    
    // split_terminator
    let with_trailing = "a,b,c,";
    let parts: Vec<&str> = with_trailing.split_terminator(',').collect();
    println!("With terminator: {:?}", parts); // ไม่มี empty string ท้าย
}
```

### trim และการ normalize whitespace

```rust
fn string_trim_methods() {
    let padded = "   hello world   ";
    
    println!("Original: '{}'", padded);
    println!("trim(): '{}'", padded.trim());
    println!("trim_start(): '{}'", padded.trim_start());
    println!("trim_end(): '{}'", padded.trim_end());
    
    // trim_matches - trim ตาม character/pattern
    let custom = "###hello###";
    println!("trim '#': '{}'", custom.trim_matches('#'));
    
    let mixed = "--hello--world--";
    println!("trim '-': '{}'", mixed.trim_matches('-'));
    
    // trim_start_matches / trim_end_matches
    let url = "https://example.com";
    let without_scheme = url.trim_start_matches("https://");
    println!("Without scheme: '{}'", without_scheme);
    
    // ลบ trailing newline
    let with_newline = "hello\n";
    let trimmed = with_newline.trim_end_matches('\n');
    println!("Without newline: '{}'", trimmed);
}
```

### replace

```rust
fn string_replace_methods() {
    let text = "I like cats. Cats are cute. I have a cat.";
    
    // replace - แทนที่ทุก occurrence
    let replaced = text.replace("cat", "dog");
    println!("Replace all: {}", replaced);
    
    // replacen - แทนที่ n ครั้งแรก
    let replaced_n = text.replacen("cat", "dog", 2);
    println!("Replace 2: {}", replaced_n);
    
    // replace case-sensitive
    let case_sensitive = text.replace("Cat", "DOG");
    println!("Case sensitive: {}", case_sensitive);
    
    // Multiple replacements ด้วย closure pattern
    let mut result = text.to_string();
    let replacements = [
        ("cat", "dog"),
        ("cute", "adorable"),
        ("like", "love"),
    ];
    
    for (from, to) in &replacements {
        result = result.replace(from, to);
    }
    println!("Multiple: {}", result);
}
```

---

## 3. String Building

### push_str และ push

```rust
fn string_building() {
    let mut s = String::new();
    
    // push_str - เพิ่ม string slice
    s.push_str("Hello");
    s.push_str(", ");
    s.push_str("World");
    println!("push_str: {}", s);
    
    // push - เพิ่ม single character
    s.push('!');
    println!("push char: {}", s);
    
    // + operator
    let s1 = String::from("Hello");
    let s2 = String::from(", World!");
    let s3 = s1 + &s2; // s1 ถูก move
    println!("Concatenated: {}", s3);
    
    // += operator
    let mut greeting = String::from("Hello");
    greeting += " World";
    println!("+=: {}", greeting);
    
    // extend - เพิ่ม iterator
    let mut s = String::from("Hello");
    s.extend([',', ' ', 'W', 'o', 'r', 'l', 'd']);
    println!("extend: {}", s);
    
    // repeat
    let star = "*".repeat(10);
    println!("repeat: {}", star);
    
    // insert / insert_str
    let mut s = String::from("Hello World");
    s.insert(5, ',');
    println!("After insert: {}", s);
    s.insert_str(0, ">>> ");
    println!("After insert_str: {}", s);
}
```

### format! และ concat!

```rust
fn string_formatting() {
    // format! - สร้าง String ด้วย formatting
    let name = "Rust";
    let version = 1.75;
    let formatted = format!("{} v{:.1}", name, version);
    println!("{}", formatted);
    
    // format! with padding
    let padded = format!("{:>20}", "right");   // right-align
    let padded2 = format!("{:<20}", "left");   // left-align
    let padded3 = format!("{:^20}", "center"); // center
    let padded4 = format!("{:0>5}", 42);       // zero-pad
    
    println!("'{}'", padded);
    println!("'{}'", padded2);
    println!("'{}'", padded3);
    println!("'{}'", padded4);
    
    // format! with precision
    let pi = std::f64::consts::PI;
    println!("{:.2}", pi);    // 3.14
    println!("{:.5}", pi);    // 3.14159
    println!("{:10.3}", pi);  // width 10, precision 3
    
    // concat! - compile-time string concat
    const HELLO: &str = concat!("Hello", ", ", "World", "!");
    println!("{}", HELLO);
    
    // Building URL strings
    let base = "https://api.example.com";
    let version = "v1";
    let endpoint = "users";
    let id = 42;
    
    let url = format!("{}/{}/{}/{}", base, version, endpoint, id);
    println!("URL: {}", url);
    
    // Template-like formatting
    let template = format!(
        "Dear {name},\n\
         Your order #{order_id} has been confirmed.\n\
         Total: ${total:.2}\n\
         Thank you!",
        name = "John",
        order_id = 12345,
        total = 99.99
    );
    println!("{}", template);
}
```

---

## 4. Parsing ด้วย FromStr

```rust
use std::str::FromStr;
use std::fmt;

// Custom type ที่ parse จาก string ได้
#[derive(Debug, PartialEq)]
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

#[derive(Debug)]
struct ParseColorError(String);

impl fmt::Display for ParseColorError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "Invalid color: {}", self.0)
    }
}

impl FromStr for Color {
    type Err = ParseColorError;
    
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        // Format: "rgb(R,G,B)" หรือ "#RRGGBB"
        let s = s.trim();
        
        if let Some(rgb) = s.strip_prefix("rgb(").and_then(|s| s.strip_suffix(')')) {
            let parts: Vec<&str> = rgb.split(',').collect();
            if parts.len() != 3 {
                return Err(ParseColorError(s.to_string()));
            }
            
            let r = parts[0].trim().parse::<u8>()
                .map_err(|_| ParseColorError(s.to_string()))?;
            let g = parts[1].trim().parse::<u8>()
                .map_err(|_| ParseColorError(s.to_string()))?;
            let b = parts[2].trim().parse::<u8>()
                .map_err(|_| ParseColorError(s.to_string()))?;
            
            Ok(Color { r, g, b })
        } else if let Some(hex) = s.strip_prefix('#') {
            if hex.len() != 6 {
                return Err(ParseColorError(s.to_string()));
            }
            
            let r = u8::from_str_radix(&hex[0..2], 16)
                .map_err(|_| ParseColorError(s.to_string()))?;
            let g = u8::from_str_radix(&hex[2..4], 16)
                .map_err(|_| ParseColorError(s.to_string()))?;
            let b = u8::from_str_radix(&hex[4..6], 16)
                .map_err(|_| ParseColorError(s.to_string()))?;
            
            Ok(Color { r, g, b })
        } else {
            Err(ParseColorError(s.to_string()))
        }
    }
}

fn main() {
    // parse primitives
    let n: i32 = "42".parse().unwrap();
    let f: f64 = "3.14".parse().unwrap();
    let b: bool = "true".parse().unwrap();
    println!("Parsed: {}, {}, {}", n, f, b);
    
    // parse custom type
    let colors = [
        "rgb(255, 0, 0)",
        "rgb(0, 128, 255)",
        "#FF8800",
        "#invalid",
        "not a color",
    ];
    
    for input in &colors {
        match input.parse::<Color>() {
            Ok(color) => println!("OK: {:?}", color),
            Err(e) => println!("Error: {}", e),
        }
    }
    
    // parse กับ collect
    let numbers: Result<Vec<i32>, _> = "1 2 3 4 5"
        .split_whitespace()
        .map(str::parse)
        .collect();
    
    println!("Numbers: {:?}", numbers.unwrap());
}
```

---

## 5. Regular Expressions ด้วย regex Crate

```toml
# Cargo.toml
[dependencies]
regex = "1"
```

```rust
use regex::Regex;

fn regex_basics() {
    // สร้าง Regex (compile pattern ครั้งเดียว)
    let re = Regex::new(r"\d+").unwrap();
    
    // is_match
    println!("Has digits: {}", re.is_match("Hello 123 World"));
    println!("Has digits: {}", re.is_match("No digits here"));
    
    // find - หา match แรก
    let text = "Price: 42.50 USD";
    if let Some(m) = re.find(text) {
        println!("Found '{}' at {}..{}", m.as_str(), m.start(), m.end());
    }
    
    // find_iter - หาทุก matches
    let numbers_text = "First: 100, Second: 200, Third: 300";
    let numbers: Vec<&str> = re.find_iter(numbers_text)
        .map(|m| m.as_str())
        .collect();
    println!("All numbers: {:?}", numbers);
    
    // captures - หา capture groups
    let email_re = Regex::new(r"(\w+)@(\w+)\.(\w+)").unwrap();
    let email = "user@example.com";
    
    if let Some(caps) = email_re.captures(email) {
        println!("Full: {}", &caps[0]);
        println!("User: {}", &caps[1]);
        println!("Domain: {}", &caps[2]);
        println!("TLD: {}", &caps[3]);
    }
}

fn regex_advanced() {
    // Named capture groups
    let re = Regex::new(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})").unwrap();
    let date = "Today is 2024-01-15";
    
    if let Some(caps) = re.captures(date) {
        println!("Year: {}", &caps["year"]);
        println!("Month: {}", &caps["month"]);
        println!("Day: {}", &caps["day"]);
    }
    
    // replace
    let text = "Hello World";
    let replaced = Regex::new(r"\b\w+\b").unwrap()
        .replace_all(text, |caps: &regex::Captures| {
            caps[0].to_uppercase()
        });
    println!("Replaced: {}", replaced);
    
    // split
    let re = Regex::new(r"\s+").unwrap();
    let parts: Vec<&str> = re.split("  hello   world   rust  ").collect();
    println!("Split: {:?}", parts);
}

fn validate_patterns() {
    // Email validation
    let email_re = Regex::new(
        r"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"
    ).unwrap();
    
    let emails = ["user@example.com", "invalid-email", "test@test.co.th"];
    for email in &emails {
        println!("{}: {}", email, if email_re.is_match(email) { "valid" } else { "invalid" });
    }
    
    // Phone validation (Thai format)
    let phone_re = Regex::new(r"^(0[0-9]{1,2}[-\s]?[0-9]{3,4}[-\s]?[0-9]{3,4})$").unwrap();
    
    let phones = ["081-234-5678", "02-123-4567", "invalid"];
    for phone in &phones {
        println!("{}: {}", phone, if phone_re.is_match(phone) { "valid" } else { "invalid" });
    }
}

fn main() {
    println!("=== Regex Basics ===");
    regex_basics();
    
    println!("\n=== Advanced Regex ===");
    regex_advanced();
    
    println!("\n=== Validation ===");
    validate_patterns();
}
```

---

## 6. Unicode Handling

```rust
fn unicode_basics() {
    let thai = "สวัสดีชาวโลก";
    let emoji = "Hello 🌍🦀 Rust!";
    let arabic = "مرحبا";
    
    // chars() - iterate over Unicode scalar values
    println!("Thai chars:");
    for (i, c) in thai.chars().enumerate() {
        println!("  [{}] U+{:04X} '{}'", i, c as u32, c);
    }
    
    // bytes() vs chars()
    println!("\nThai bytes: {}", thai.len());        // byte count
    println!("Thai chars: {}", thai.chars().count()); // char count
    
    // char_indices() - index และ char
    println!("\nEmoji indices:");
    for (byte_pos, char) in emoji.char_indices() {
        println!("  byte {} = '{}'", byte_pos, char);
    }
    
    // String slicing ต้องระวัง
    // let wrong = &thai[0..1];  // PANIC! ตัดกลาง UTF-8 sequence
    
    // วิธีที่ถูกต้อง - ใช้ char boundaries
    let first_char = thai.chars().next().unwrap();
    let first_char_len = first_char.len_utf8();
    let first: &str = &thai[..first_char_len];
    println!("\nFirst char: {}", first);
    
    // nth char
    let third = thai.chars().nth(2);
    println!("Third char: {:?}", third);
}

fn unicode_operations() {
    // Case conversion
    let mixed = "Hello WORLD rust";
    println!("to_uppercase: {}", mixed.to_uppercase());
    println!("to_lowercase: {}", mixed.to_lowercase());
    
    // Unicode-aware case (ภาษาอื่น)
    let german = "straße";  // German sharp s
    println!("Uppercase: {}", german.to_uppercase()); // STRASSE
    
    // char classification
    let test_chars = ['A', 'a', '1', ' ', '!', 'ก', '中', '🦀'];
    for c in &test_chars {
        println!("'{}': alphabetic={}, numeric={}, whitespace={}, ascii={}",
            c,
            c.is_alphabetic(),
            c.is_numeric(),
            c.is_whitespace(),
            c.is_ascii()
        );
    }
    
    // Normalization (requires unicode-normalization crate)
    // Use NFC, NFD, NFKC, NFKD forms for comparison
}

fn grapheme_clusters() {
    // ต้องใช้ unicode-segmentation crate สำหรับ grapheme clusters
    // grapheme cluster คือ สิ่งที่ user มองว่าเป็น "ตัวอักษรหนึ่ง"
    
    let text = "é"; // e + combining accent = 1 grapheme, 2 chars
    println!("Chars: {}", text.chars().count()); // อาจเป็น 1 หรือ 2
    
    // Thai with vowel marks
    let thai_word = "กา"; // ก + า = 1 grapheme
    println!("Thai chars: {}", thai_word.chars().count());
    
    // สำหรับ grapheme cluster ที่ถูกต้อง ใช้:
    // use unicode_segmentation::UnicodeSegmentation;
    // let graphemes: Vec<&str> = text.graphemes(true).collect();
}

fn main() {
    println!("=== Unicode Basics ===");
    unicode_basics();
    
    println!("\n=== Unicode Operations ===");
    unicode_operations();
    
    println!("\n=== Grapheme Clusters ===");
    grapheme_clusters();
}
```

---

## 7. String Encoding/Decoding

```rust
fn encoding_basics() {
    // UTF-8 encoding/decoding
    let text = "Hello, สวัสดี!";
    
    // String เป็น UTF-8 เสมอ
    let bytes = text.as_bytes();
    println!("Bytes: {:?}", &bytes[..10]); // first 10 bytes
    
    // decode จาก bytes
    match std::str::from_utf8(bytes) {
        Ok(s) => println!("Valid UTF-8: {}", s),
        Err(e) => println!("Invalid UTF-8: {}", e),
    }
    
    // invalid UTF-8
    let invalid_bytes = vec![0xFF, 0xFE, 0x41]; // invalid sequence
    match std::str::from_utf8(&invalid_bytes) {
        Ok(s) => println!("Valid: {}", s),
        Err(e) => println!("Invalid at position: {}", e.valid_up_to()),
    }
    
    // lossy conversion
    let lossy = String::from_utf8_lossy(&invalid_bytes);
    println!("Lossy: {}", lossy); // แทนที่ invalid ด้วย �
    
    // OsString สำหรับ OS paths
    use std::ffi::OsString;
    let os_string = OsString::from("file.txt");
    if let Some(s) = os_string.to_str() {
        println!("OsString as str: {}", s);
    }
}

fn base64_example() {
    // base64 encoding (ต้องใช้ base64 crate)
    let data = b"Hello, World!";
    
    // manual base64 encode (simplified)
    let alphabet = b"ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
    let mut result = String::new();
    
    for chunk in data.chunks(3) {
        let b0 = chunk[0] as u32;
        let b1 = if chunk.len() > 1 { chunk[1] as u32 } else { 0 };
        let b2 = if chunk.len() > 2 { chunk[2] as u32 } else { 0 };
        
        let combined = (b0 << 16) | (b1 << 8) | b2;
        
        result.push(alphabet[(combined >> 18) as usize] as char);
        result.push(alphabet[((combined >> 12) & 0x3F) as usize] as char);
        
        if chunk.len() > 1 {
            result.push(alphabet[((combined >> 6) & 0x3F) as usize] as char);
        } else {
            result.push('=');
        }
        
        if chunk.len() > 2 {
            result.push(alphabet[(combined & 0x3F) as usize] as char);
        } else {
            result.push('=');
        }
    }
    
    println!("Base64: {}", result);
}

fn main() {
    println!("=== Encoding Basics ===");
    encoding_basics();
    
    println!("\n=== Base64 ===");
    base64_example();
}
```

---

## 8. Practical: Text Processing Utilities

```rust
use std::collections::HashMap;

/// นับความถี่ของคำ
fn word_frequency(text: &str) -> HashMap<String, usize> {
    let mut freq = HashMap::new();
    
    for word in text.split_whitespace() {
        // normalize: lowercase, remove punctuation
        let clean_word: String = word.chars()
            .filter(|c| c.is_alphabetic())
            .collect::<String>()
            .to_lowercase();
        
        if !clean_word.is_empty() {
            *freq.entry(clean_word).or_insert(0) += 1;
        }
    }
    
    freq
}

/// หาคำที่ใช้บ่อยที่สุด
fn top_words(freq: &HashMap<String, usize>, n: usize) -> Vec<(&str, usize)> {
    let mut pairs: Vec<(&str, usize)> = freq.iter()
        .map(|(k, &v)| (k.as_str(), v))
        .collect();
    
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    pairs.into_iter().take(n).collect()
}

/// Caesar cipher (simple encryption)
fn caesar_cipher(text: &str, shift: u8) -> String {
    text.chars().map(|c| {
        if c.is_ascii_alphabetic() {
            let base = if c.is_uppercase() { b'A' } else { b'a' };
            let shifted = (c as u8 - base + shift) % 26 + base;
            shifted as char
        } else {
            c
        }
    }).collect()
}

/// แปลง camelCase เป็น snake_case
fn camel_to_snake(s: &str) -> String {
    let mut result = String::new();
    let mut chars = s.chars().peekable();
    
    while let Some(c) = chars.next() {
        if c.is_uppercase() && !result.is_empty() {
            // ใส่ underscore ก่อน uppercase (ยกเว้นตอนเริ่ม)
            // ไม่ใส่ถ้า previous char เป็น uppercase ด้วย
            let prev_was_upper = result.chars().last()
                .map(|prev| prev.is_uppercase())
                .unwrap_or(false);
            
            if !prev_was_upper {
                result.push('_');
            }
        }
        result.push(c.to_lowercase().next().unwrap());
    }
    
    result
}

/// แปลง snake_case เป็น camelCase
fn snake_to_camel(s: &str) -> String {
    let mut capitalize_next = false;
    let mut result = String::new();
    
    for c in s.chars() {
        if c == '_' {
            capitalize_next = true;
        } else if capitalize_next {
            result.push(c.to_uppercase().next().unwrap());
            capitalize_next = false;
        } else {
            result.push(c);
        }
    }
    
    result
}

/// Wrap text ตามความกว้างที่กำหนด
fn word_wrap(text: &str, width: usize) -> String {
    let mut result = String::new();
    let mut current_line_len = 0;
    
    for word in text.split_whitespace() {
        let word_len = word.chars().count();
        
        if current_line_len + word_len + 1 > width && current_line_len > 0 {
            result.push('\n');
            current_line_len = 0;
        } else if current_line_len > 0 {
            result.push(' ');
            current_line_len += 1;
        }
        
        result.push_str(word);
        current_line_len += word_len;
    }
    
    result
}

/// สร้าง slug จาก title
fn slugify(title: &str) -> String {
    title.to_lowercase()
        .chars()
        .map(|c| if c.is_alphanumeric() { c } else { '-' })
        .collect::<String>()
        .split('-')
        .filter(|s| !s.is_empty())
        .collect::<Vec<_>>()
        .join("-")
}

/// Extract links จาก markdown text
fn extract_markdown_links(text: &str) -> Vec<(String, String)> {
    let mut links = Vec::new();
    let mut chars = text.chars().peekable();
    
    while let Some(c) = chars.next() {
        if c == '[' {
            // อ่าน link text
            let mut link_text = String::new();
            let mut found_close = false;
            
            for inner in chars.by_ref() {
                if inner == ']' {
                    found_close = true;
                    break;
                }
                link_text.push(inner);
            }
            
            if found_close {
                // ตรวจสอบ URL ใน ()
                if chars.peek() == Some(&'(') {
                    chars.next(); // consume '('
                    let mut url = String::new();
                    
                    for inner in chars.by_ref() {
                        if inner == ')' {
                            break;
                        }
                        url.push(inner);
                    }
                    
                    links.push((link_text, url));
                }
            }
        }
    }
    
    links
}

/// Simple template engine
fn render_template(template: &str, vars: &HashMap<&str, &str>) -> String {
    let mut result = template.to_string();
    
    for (key, value) in vars {
        let placeholder = format!("{{{{{}}}}}", key);
        result = result.replace(&placeholder, value);
    }
    
    result
}

fn main() {
    println!("=== Word Frequency ===");
    let text = "the quick brown fox jumps over the lazy dog the fox";
    let freq = word_frequency(text);
    let top = top_words(&freq, 5);
    println!("Top 5 words:");
    for (word, count) in top {
        println!("  '{}': {}", word, count);
    }
    
    println!("\n=== Caesar Cipher ===");
    let message = "Hello, World!";
    let encrypted = caesar_cipher(message, 3);
    let decrypted = caesar_cipher(&encrypted, 23); // 26 - 3
    println!("Original: {}", message);
    println!("Encrypted: {}", encrypted);
    println!("Decrypted: {}", decrypted);
    
    println!("\n=== Case Conversion ===");
    let camel = "helloWorldRustLang";
    let snake = camel_to_snake(camel);
    println!("camelCase: {}", camel);
    println!("snake_case: {}", snake);
    println!("Back to camelCase: {}", snake_to_camel(&snake));
    
    println!("\n=== Word Wrap ===");
    let long_text = "The quick brown fox jumps over the lazy dog. This is a long text that needs to be wrapped.";
    println!("Wrapped at 40 chars:");
    println!("{}", word_wrap(long_text, 40));
    
    println!("\n=== Slugify ===");
    let titles = ["Hello World!", "Rust is Awesome", "สวัสดี Rust", "My Blog Post - 2024"];
    for title in &titles {
        println!("'{}' -> '{}'", title, slugify(title));
    }
    
    println!("\n=== Extract Links ===");
    let markdown = "Visit [Rust](https://www.rust-lang.org) and [Cargo](https://crates.io) for more info.";
    let links = extract_markdown_links(markdown);
    for (text, url) in links {
        println!("Text: '{}', URL: '{}'", text, url);
    }
    
    println!("\n=== Template Rendering ===");
    let template = "Hello, {{name}}! You have {{count}} messages. Your role is {{role}}.";
    let mut vars = HashMap::new();
    vars.insert("name", "Alice");
    vars.insert("count", "5");
    vars.insert("role", "Administrator");
    println!("{}", render_template(template, &vars));
}
```

---

## 9. String Parsing Utilities

```rust
/// Parse key-value pairs จาก string
fn parse_key_value(input: &str) -> HashMap<String, String> {
    input.lines()
        .filter_map(|line| {
            let line = line.trim();
            if line.is_empty() || line.starts_with('#') {
                return None;
            }
            
            let (key, value) = line.split_once('=')?;
            Some((
                key.trim().to_string(),
                value.trim().trim_matches('"').to_string(),
            ))
        })
        .collect()
}

/// Parse CSV line
fn parse_csv_line(line: &str) -> Vec<String> {
    let mut fields = Vec::new();
    let mut current = String::new();
    let mut in_quotes = false;
    let mut chars = line.chars().peekable();
    
    while let Some(c) = chars.next() {
        match c {
            '"' => {
                if in_quotes && chars.peek() == Some(&'"') {
                    chars.next(); // skip escaped quote
                    current.push('"');
                } else {
                    in_quotes = !in_quotes;
                }
            }
            ',' if !in_quotes => {
                fields.push(current.trim().to_string());
                current.clear();
            }
            _ => current.push(c),
        }
    }
    fields.push(current.trim().to_string());
    fields
}

fn main() {
    println!("=== Parse Key-Value ===");
    let config = r#"
# Database configuration
host = localhost
port = 5432
database = "mydb"
user = admin
password = "secret123"
    "#;
    
    let kv = parse_key_value(config);
    for (key, value) in &kv {
        println!("  {} = {}", key, value);
    }
    
    println!("\n=== Parse CSV ===");
    let csv_lines = [
        r#"John,30,"New York, NY",Engineer"#,
        r#"Jane,25,"Los Angeles","Product Manager""#,
        r#""Bob Smith",35,Seattle,Developer"#,
    ];
    
    for line in &csv_lines {
        let fields = parse_csv_line(line);
        println!("Fields: {:?}", fields);
    }
}
```

---

## สรุป

| Operation | Method/Function |
|-----------|----------------|
| Search | `contains`, `find`, `rfind` |
| Check prefix/suffix | `starts_with`, `ends_with` |
| Split | `split`, `splitn`, `split_whitespace`, `lines` |
| Trim | `trim`, `trim_start`, `trim_end`, `trim_matches` |
| Replace | `replace`, `replacen` |
| Build | `push_str`, `push`, `format!`, `+` operator |
| Parse | `.parse::<T>()`, `FromStr` trait |
| Case | `to_uppercase`, `to_lowercase` |
| Unicode | `chars()`, `char_indices()`, `bytes()` |
| Check | `is_empty`, `len`, `contains` |

---

## Navigation

- [← Part 018: Advanced Cargo and Tooling](../part_018/README.md)
- [→ Part 020: Procedural Macros and derive](../part_020/README.md)
- [กลับหน้าหลัก](../../README.md)
