# Part 006: Pattern Matching และ Option/Result ขั้นสูง 🎯

## 🎯 เป้าหมายของ Part นี้

- Pattern matching ขั้นสูงทุกรูปแบบ
- Nested patterns
- Guards และ Bindings
- Error handling patterns

---

## 1. Pattern Matching ขั้นสูง

### 1.1 All Pattern Types

```rust
fn main() {
    // 1. Literal patterns
    let x = 5;
    match x {
        1 => println!("one"),
        2 => println!("two"),
        3..=5 => println!("three to five"),
        _ => println!("other"),
    }

    // 2. Named variable patterns
    let y = 10;
    match y {
        n if n < 0 => println!("negative: {}", n),
        0 => println!("zero"),
        n => println!("positive: {}", n),
    }

    // 3. Multiple patterns with |
    let z = 3;
    match z {
        1 | 2 => println!("one or two"),
        3 | 4 => println!("three or four"),
        _ => println!("other"),
    }

    // 4. Range patterns
    let score = 75;
    let grade = match score {
        90..=100 => 'A',
        80..=89  => 'B',
        70..=79  => 'C',
        60..=69  => 'D',
        0..=59   => 'F',
        _        => '?',
    };
    println!("Grade: {}", grade);

    // 5. Tuple patterns
    let pair = (true, false);
    match pair {
        (true, true)   => println!("Both true"),
        (true, false)  => println!("First true"),
        (false, true)  => println!("Second true"),
        (false, false) => println!("Both false"),
    }

    // 6. Struct patterns
    struct Point { x: i32, y: i32 }
    let p = Point { x: 3, y: 0 };

    match p {
        Point { x: 0, y: 0 } => println!("Origin"),
        Point { x, y: 0 }    => println!("On x-axis: {}", x),
        Point { x: 0, y }    => println!("On y-axis: {}", y),
        Point { x, y }       => println!("Point: ({}, {})", x, y),
    }

    // 7. @ bindings
    let n = 15;
    match n {
        x @ 1..=10  => println!("1-10: {}", x),
        x @ 11..=20 => println!("11-20: {}", x),
        x           => println!("other: {}", x),
    }

    // 8. Ignore with _
    let (a, _, c) = (1, 2, 3);
    println!("a={}, c={}", a, c);

    // 9. .. to ignore remaining
    struct Config { host: String, port: u16, debug: bool, verbose: bool }
    let config = Config {
        host: "localhost".to_string(),
        port: 8080,
        debug: true,
        verbose: false,
    };

    let Config { host, port, .. } = config;
    println!("Connecting to {}:{}", host, port);

    // 10. Nested patterns
    let nested = Some(Some(42));
    match nested {
        Some(Some(n)) if n > 0 => println!("Positive nested: {}", n),
        Some(Some(n))           => println!("Nested: {}", n),
        Some(None)              => println!("Inner None"),
        None                    => println!("Outer None"),
    }
}
```

### 1.2 Matching Enums with Complex Data

```rust
#[derive(Debug)]
enum Json {
    Null,
    Bool(bool),
    Number(f64),
    Str(String),
    Array(Vec<Json>),
    Object(std::collections::HashMap<String, Json>),
}

impl Json {
    fn type_name(&self) -> &str {
        match self {
            Json::Null       => "null",
            Json::Bool(_)    => "boolean",
            Json::Number(_)  => "number",
            Json::Str(_)     => "string",
            Json::Array(_)   => "array",
            Json::Object(_)  => "object",
        }
    }

    fn is_truthy(&self) -> bool {
        match self {
            Json::Null           => false,
            Json::Bool(b)        => *b,
            Json::Number(n)      => *n != 0.0,
            Json::Str(s)         => !s.is_empty(),
            Json::Array(arr)     => !arr.is_empty(),
            Json::Object(obj)    => !obj.is_empty(),
        }
    }

    fn as_number(&self) -> Option<f64> {
        match self {
            Json::Number(n) => Some(*n),
            Json::Bool(b) => Some(if *b { 1.0 } else { 0.0 }),
            Json::Str(s) => s.parse().ok(),
            _ => None,
        }
    }
}

impl std::fmt::Display for Json {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        match self {
            Json::Null       => write!(f, "null"),
            Json::Bool(b)    => write!(f, "{}", b),
            Json::Number(n)  => write!(f, "{}", n),
            Json::Str(s)     => write!(f, "\"{}\"", s),
            Json::Array(arr) => {
                write!(f, "[")?;
                for (i, v) in arr.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "{}", v)?;
                }
                write!(f, "]")
            },
            Json::Object(obj) => {
                write!(f, "{{")?;
                for (i, (k, v)) in obj.iter().enumerate() {
                    if i > 0 { write!(f, ", ")?; }
                    write!(f, "\"{}\": {}", k, v)?;
                }
                write!(f, "}}")
            },
        }
    }
}

fn main() {
    use std::collections::HashMap;

    let values = vec![
        Json::Null,
        Json::Bool(true),
        Json::Bool(false),
        Json::Number(42.0),
        Json::Str(String::from("hello")),
        Json::Array(vec![
            Json::Number(1.0),
            Json::Number(2.0),
            Json::Number(3.0),
        ]),
    ];

    for v in &values {
        println!("{}: type={}, truthy={}, as_num={:?}",
            v, v.type_name(), v.is_truthy(), v.as_number()
        );
    }
}
```

---

## 2. Error Handling Patterns

### 2.1 Custom Error Types

```rust
use std::fmt;
use std::num::ParseIntError;

// Custom error enum
#[derive(Debug)]
enum MathError {
    DivisionByZero,
    NegativeSquareRoot(f64),
    Overflow,
    ParseError(ParseIntError),
}

impl fmt::Display for MathError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            MathError::DivisionByZero =>
                write!(f, "ไม่สามารถหารด้วยศูนย์ได้"),
            MathError::NegativeSquareRoot(n) =>
                write!(f, "ไม่สามารถหารากที่สองของ {} ได้ (ค่าลบ)", n),
            MathError::Overflow =>
                write!(f, "ค่าเกิน overflow"),
            MathError::ParseError(e) =>
                write!(f, "Parse error: {}", e),
        }
    }
}

impl From<ParseIntError> for MathError {
    fn from(e: ParseIntError) -> Self {
        MathError::ParseError(e)
    }
}

fn safe_divide(a: f64, b: f64) -> Result<f64, MathError> {
    if b == 0.0 {
        Err(MathError::DivisionByZero)
    } else {
        Ok(a / b)
    }
}

fn safe_sqrt(n: f64) -> Result<f64, MathError> {
    if n < 0.0 {
        Err(MathError::NegativeSquareRoot(n))
    } else {
        Ok(n.sqrt())
    }
}

fn parse_and_compute(a: &str, b: &str) -> Result<f64, MathError> {
    let x: i32 = a.parse()?;  // ? converts ParseIntError via From impl
    let y: i32 = b.parse()?;
    safe_divide(x as f64, y as f64)
}

fn main() {
    // Test cases
    let cases = vec![
        (10.0, 2.0),
        (10.0, 0.0),
        (-1.0, 0.0),
    ];

    for (a, b) in &cases {
        match safe_divide(*a, *b) {
            Ok(result) => println!("{} / {} = {}", a, b, result),
            Err(e) => println!("{} / {} = Error: {}", a, b, e),
        }
    }

    let sqrt_cases = [4.0, 9.0, -1.0, 0.0];
    for &n in &sqrt_cases {
        match safe_sqrt(n) {
            Ok(r) => println!("√{} = {}", n, r),
            Err(e) => println!("√{} = Error: {}", n, e),
        }
    }

    // parse_and_compute
    println!("{:?}", parse_and_compute("10", "3"));
    println!("{:?}", parse_and_compute("10", "0"));
    println!("{:?}", parse_and_compute("abc", "3"));
}
```

### 2.2 Error Handling Best Practices

```rust
use std::fs;
use std::io;

// ใช้ thiserror crate (ยอดนิยม) ในโปรเจกต์จริง
// Cargo.toml: thiserror = "1"

// สำหรับตอนนี้ implement เอง
#[derive(Debug)]
enum ConfigError {
    FileNotFound(String),
    ParseError(String),
    InvalidValue { field: String, value: String },
    MissingField(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::FileNotFound(path) =>
                write!(f, "Config file not found: {}", path),
            ConfigError::ParseError(msg) =>
                write!(f, "Parse error: {}", msg),
            ConfigError::InvalidValue { field, value } =>
                write!(f, "Invalid value '{}' for field '{}'", value, field),
            ConfigError::MissingField(field) =>
                write!(f, "Missing required field: {}", field),
        }
    }
}

#[derive(Debug)]
struct Config {
    host: String,
    port: u16,
    debug: bool,
}

impl Config {
    fn from_str(content: &str) -> Result<Self, ConfigError> {
        let mut host = None;
        let mut port = None;
        let mut debug = false;

        for line in content.lines() {
            let line = line.trim();
            if line.is_empty() || line.starts_with('#') {
                continue;
            }

            let parts: Vec<&str> = line.splitn(2, '=').collect();
            if parts.len() != 2 {
                return Err(ConfigError::ParseError(
                    format!("Invalid line: {}", line)
                ));
            }

            let key = parts[0].trim();
            let value = parts[1].trim();

            match key {
                "host" => host = Some(value.to_string()),
                "port" => {
                    let p: u16 = value.parse().map_err(|_| {
                        ConfigError::InvalidValue {
                            field: "port".to_string(),
                            value: value.to_string(),
                        }
                    })?;
                    port = Some(p);
                },
                "debug" => {
                    debug = match value {
                        "true" | "1" | "yes" => true,
                        "false" | "0" | "no" => false,
                        _ => return Err(ConfigError::InvalidValue {
                            field: "debug".to_string(),
                            value: value.to_string(),
                        }),
                    };
                },
                _ => {} // ignore unknown keys
            }
        }

        Ok(Config {
            host: host.ok_or_else(|| ConfigError::MissingField("host".to_string()))?,
            port: port.ok_or_else(|| ConfigError::MissingField("port".to_string()))?,
            debug,
        })
    }
}

fn main() {
    let valid_config = "
        # Server configuration
        host = localhost
        port = 8080
        debug = true
    ";

    match Config::from_str(valid_config) {
        Ok(config) => println!("Config: {:?}", config),
        Err(e) => println!("Error: {}", e),
    }

    let invalid_config = "
        host = localhost
        port = abc
    ";

    match Config::from_str(invalid_config) {
        Ok(config) => println!("Config: {:?}", config),
        Err(e) => println!("Error: {}", e),
    }

    let missing_config = "
        host = localhost
    ";

    match Config::from_str(missing_config) {
        Ok(config) => println!("Config: {:?}", config),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## 3. สรุปและ Exercises

### 3.1 สิ่งที่เรียนรู้

✅ Pattern types ทั้งหมด  
✅ Guards และ @ bindings  
✅ Matching nested structures  
✅ Custom Error types  
✅ From trait สำหรับ error conversion  
✅ ? operator ขั้นสูง  

### 3.2 Exercise

**Exercise: Expression Evaluator**
```rust
enum Expr {
    Num(f64),
    Add(Box<Expr>, Box<Expr>),
    Sub(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
    Div(Box<Expr>, Box<Expr>),
}

// Implement fn eval(expr: &Expr) -> Result<f64, MathError>
```

---

*[← Part 005: Structs และ Enums](../part_005/README.md) | [Part 007: Collections →](../part_007/README.md)*
