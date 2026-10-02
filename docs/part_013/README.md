# Part 013: Testing ใน Rust 🧪

## 🎯 เป้าหมายของ Part นี้

- Unit tests และ test macros
- Integration tests ใน `tests/` directory
- Test organization และ `#[cfg(test)]`
- `#[should_panic]` และ Result ใน tests
- Test fixtures และ setup/teardown
- Mocking ด้วย mockall crate
- Property-based testing ด้วย proptest
- Benchmark tests ด้วย Criterion

---

## 1. Unit Tests พื้นฐาน

```rust
// src/calculator.rs
pub fn add(a: i32, b: i32) -> i32 { a + b }
pub fn subtract(a: i32, b: i32) -> i32 { a - b }
pub fn multiply(a: i32, b: i32) -> i32 { a * b }
pub fn divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

pub fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

pub fn is_palindrome(s: &str) -> bool {
    let cleaned: String = s.chars()
        .filter(|c| c.is_alphanumeric())
        .map(|c| c.to_lowercase().next().unwrap())
        .collect();
    cleaned == cleaned.chars().rev().collect::<String>()
}

// Unit tests อยู่ใน module เดียวกับ code ที่ test
#[cfg(test)]
mod tests {
    // super:: import ทุกอย่างจาก parent module
    use super::*;

    // ทุก test function ต้องมี #[test] attribute
    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
        assert_eq!(add(-1, 1), 0);
        assert_eq!(add(0, 0), 0);
    }

    #[test]
    fn test_subtract() {
        assert_eq!(subtract(10, 3), 7);
        assert_eq!(subtract(0, 5), -5);
    }

    #[test]
    fn test_multiply() {
        assert_eq!(multiply(4, 5), 20);
        assert_eq!(multiply(-2, 3), -6);
        assert_eq!(multiply(0, 100), 0);
    }

    #[test]
    fn test_divide() {
        assert_eq!(divide(10.0, 2.0), Some(5.0));
        assert_eq!(divide(7.0, 0.0), None);
        
        // Float comparison ต้องระวัง
        let result = divide(1.0, 3.0).unwrap();
        assert!((result - 0.3333333).abs() < 1e-6);
    }

    #[test]
    fn test_fibonacci() {
        assert_eq!(fibonacci(0), 0);
        assert_eq!(fibonacci(1), 1);
        assert_eq!(fibonacci(10), 55);
        assert_eq!(fibonacci(20), 6765);
    }

    #[test]
    fn test_is_palindrome() {
        assert!(is_palindrome("racecar"));
        assert!(is_palindrome("A man a plan a canal Panama"));
        assert!(is_palindrome("Was it a car or a cat I saw"));
        assert!(!is_palindrome("hello"));
        assert!(!is_palindrome("Rust"));
    }
}
```

---

## 2. Assert Macros

```rust
#[cfg(test)]
mod assert_examples {
    // assert! - ตรวจสอบว่า expression เป็น true
    #[test]
    fn test_assert() {
        assert!(2 + 2 == 4);
        assert!(true);
        assert!(vec![1, 2, 3].len() == 3);
        
        // with custom message
        let x = 42;
        assert!(x > 0, "x should be positive, got {}", x);
    }

    // assert_eq! - ตรวจสอบว่า 2 ค่าเท่ากัน
    #[test]
    fn test_assert_eq() {
        assert_eq!(2 + 2, 4);
        assert_eq!("hello".to_uppercase(), "HELLO");
        assert_eq!(vec![1, 2, 3], vec![1, 2, 3]);
        
        // with custom message
        let result = 2 * 3;
        assert_eq!(result, 6, "Expected 6, got {}", result);
    }

    // assert_ne! - ตรวจสอบว่า 2 ค่าไม่เท่ากัน
    #[test]
    fn test_assert_ne() {
        assert_ne!(1, 2);
        assert_ne!("hello", "world");
        
        let empty: Vec<i32> = Vec::new();
        let non_empty = vec![1, 2, 3];
        assert_ne!(empty, non_empty);
    }

    // Comparing floats
    #[test]
    fn test_float_comparison() {
        let a = 0.1 + 0.2;
        let b = 0.3;
        
        // Direct comparison often fails due to floating point
        // assert_eq!(a, b);  // might fail!
        
        // Better: use epsilon comparison
        assert!((a - b).abs() < f64::EPSILON * 10.0,
            "Expected {} ≈ {}", a, b);
        
        // Or use approx crate
        // assert_approx_eq!(a, b, 1e-10);
    }

    // Testing with complex types
    #[derive(Debug, PartialEq)]
    struct Point {
        x: f64,
        y: f64,
    }

    #[test]
    fn test_struct_equality() {
        let p1 = Point { x: 1.0, y: 2.0 };
        let p2 = Point { x: 1.0, y: 2.0 };
        let p3 = Point { x: 3.0, y: 4.0 };
        
        assert_eq!(p1, p2);
        assert_ne!(p1, p3);
    }
}
```

---

## 3. should_panic

```rust
pub fn parse_positive(s: &str) -> u32 {
    let n: i32 = s.parse().expect("Invalid number");
    if n < 0 {
        panic!("Number must be positive, got {}", n);
    }
    n as u32
}

pub fn get_element(v: &[i32], index: usize) -> i32 {
    if index >= v.len() {
        panic!("Index {} out of bounds for length {}", index, v.len());
    }
    v[index]
}

#[cfg(test)]
mod panic_tests {
    use super::*;

    // should_panic - test ต้อง panic เพื่อผ่าน
    #[test]
    #[should_panic]
    fn test_negative_panics() {
        parse_positive("-5");
    }

    // should_panic with expected message
    #[test]
    #[should_panic(expected = "Number must be positive")]
    fn test_negative_panics_with_message() {
        parse_positive("-10");
    }

    // should_panic กับ out of bounds
    #[test]
    #[should_panic(expected = "out of bounds")]
    fn test_out_of_bounds() {
        let v = vec![1, 2, 3];
        get_element(&v, 10);
    }

    // Test ที่ไม่ panic (normal test)
    #[test]
    fn test_valid_positive() {
        assert_eq!(parse_positive("42"), 42);
        assert_eq!(parse_positive("0"), 0);
    }
}
```

---

## 4. Result ใน Tests

```rust
use std::num::ParseIntError;

pub fn parse_and_double(s: &str) -> Result<i32, ParseIntError> {
    let n = s.parse::<i32>()?;
    Ok(n * 2)
}

pub fn read_first_line(content: &str) -> Result<&str, &str> {
    content.lines()
        .next()
        .ok_or("Empty content")
}

#[cfg(test)]
mod result_tests {
    use super::*;

    // Test ที่ return Result - ถ้า return Err test จะ fail
    #[test]
    fn test_parse_double() -> Result<(), Box<dyn std::error::Error>> {
        let result = parse_and_double("21")?;
        assert_eq!(result, 42);
        
        let result2 = parse_and_double("10")?;
        assert_eq!(result2, 20);
        
        Ok(())
    }

    #[test]
    fn test_parse_error() {
        let result = parse_and_double("not_a_number");
        assert!(result.is_err());
    }

    #[test]
    fn test_read_first_line() -> Result<(), &'static str> {
        let content = "First line\nSecond line\nThird line";
        let first = read_first_line(content)?;
        assert_eq!(first, "First line");
        Ok(())
    }

    #[test]
    fn test_read_empty() {
        let result = read_first_line("");
        assert_eq!(result, Err("Empty content"));
    }

    // Test with complex error handling
    #[test]
    fn test_multiple_operations() -> Result<(), String> {
        let numbers = vec!["1", "2", "3", "4", "5"];
        
        let sum: i32 = numbers.iter()
            .map(|s| s.parse::<i32>().map_err(|e| e.to_string()))
            .collect::<Result<Vec<i32>, String>>()?
            .iter()
            .sum();
        
        assert_eq!(sum, 15);
        Ok(())
    }
}
```

---

## 5. Test Organization

```rust
// src/lib.rs
pub mod math;
pub mod strings;
pub mod collections;

// Each module has its own tests
// math.rs
pub fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 { let t = b; b = a % b; a = t; }
    a
}

pub fn lcm(a: u64, b: u64) -> u64 {
    a / gcd(a, b) * b
}

pub fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let sqrt = (n as f64).sqrt() as u64;
    (3..=sqrt).step_by(2).all(|i| n % i != 0)
}

#[cfg(test)]
mod tests {
    use super::*;
    
    // Organize related tests with nested modules
    mod gcd_tests {
        use super::*;
        
        #[test]
        fn basic() {
            assert_eq!(gcd(12, 8), 4);
            assert_eq!(gcd(100, 75), 25);
        }
        
        #[test]
        fn with_primes() {
            assert_eq!(gcd(7, 13), 1);
            assert_eq!(gcd(17, 19), 1);
        }
        
        #[test]
        fn with_zero() {
            assert_eq!(gcd(0, 5), 5);
            assert_eq!(gcd(5, 0), 5);
        }
    }
    
    mod prime_tests {
        use super::*;
        
        #[test]
        fn small_primes() {
            assert!(is_prime(2));
            assert!(is_prime(3));
            assert!(is_prime(5));
            assert!(is_prime(7));
        }
        
        #[test]
        fn composites() {
            assert!(!is_prime(4));
            assert!(!is_prime(9));
            assert!(!is_prime(15));
        }
        
        #[test]
        fn edge_cases() {
            assert!(!is_prime(0));
            assert!(!is_prime(1));
            assert!(is_prime(97));
        }
    }
    
    // Test helper functions
    fn primes_in_range(start: u64, end: u64) -> Vec<u64> {
        (start..=end).filter(|&n| is_prime(n)).collect()
    }
    
    #[test]
    fn test_primes_in_range() {
        assert_eq!(primes_in_range(1, 20), vec![2, 3, 5, 7, 11, 13, 17, 19]);
    }
    
    #[test]
    fn test_lcm() {
        assert_eq!(lcm(4, 6), 12);
        assert_eq!(lcm(12, 18), 36);
    }
}
```

---

## 6. Integration Tests

```
src/
├── lib.rs
└── calculator.rs
tests/
├── integration_test.rs
├── math_tests.rs
└── common/
    └── mod.rs    (shared test helpers)
```

```rust
// tests/common/mod.rs - shared test utilities
pub fn setup_test_data() -> Vec<i32> {
    vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
}

pub struct TestContext {
    pub data: Vec<i32>,
    pub multiplier: i32,
}

impl TestContext {
    pub fn new() -> Self {
        TestContext {
            data: setup_test_data(),
            multiplier: 2,
        }
    }
    
    pub fn with_multiplier(multiplier: i32) -> Self {
        TestContext {
            data: setup_test_data(),
            multiplier,
        }
    }
}
```

```rust
// tests/integration_test.rs
// Integration tests ใช้ library crate เหมือน external user
// ดังนั้นต้อง import ด้วย crate name

mod common;  // import shared test utilities

// Assume our library is called `my_math`
// use my_math::{gcd, lcm, is_prime};

// Mock implementation for demonstration
fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 { let t = b; b = a % b; a = t; }
    a
}

fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }
    let sqrt = (n as f64).sqrt() as u64;
    (3..=sqrt).step_by(2).all(|i| n % i != 0)
}

#[test]
fn test_library_gcd_workflow() {
    // Integration test: test full workflow
    let numbers = vec![12u64, 18, 24, 30];
    
    let overall_gcd = numbers.iter()
        .copied()
        .reduce(gcd)
        .unwrap();
    
    assert_eq!(overall_gcd, 6);
}

#[test]
fn test_prime_counting() {
    let primes: Vec<u64> = (2..100).filter(|&n| is_prime(n)).collect();
    
    // There are 25 primes below 100
    assert_eq!(primes.len(), 25);
    assert_eq!(primes[0], 2);
    assert_eq!(primes[primes.len() - 1], 97);
}

#[test]
fn test_with_common_data() {
    let ctx = common::TestContext::new();
    
    let sum: i32 = ctx.data.iter()
        .map(|&x| x * ctx.multiplier)
        .sum();
    
    assert_eq!(sum, 110);  // (1+2+...+10) * 2 = 55 * 2 = 110
}
```

---

## 7. Test Fixtures และ Setup/Teardown

```rust
// Rust ไม่มี built-in before/after hooks แต่เราใช้ patterns เหล่านี้:

use std::sync::Once;
use std::sync::Mutex;

// Pattern 1: Setup ใน test function
#[cfg(test)]
mod fixture_patterns {
    // Test data ที่ทุก test ต้องใช้
    fn create_test_user() -> TestUser {
        TestUser {
            id: 1,
            name: "Test User".to_string(),
            email: "test@example.com".to_string(),
        }
    }
    
    #[derive(Debug, PartialEq)]
    struct TestUser {
        id: u32,
        name: String,
        email: String,
    }
    
    impl TestUser {
        fn update_email(&mut self, email: &str) {
            self.email = email.to_string();
        }
        
        fn is_valid(&self) -> bool {
            !self.name.is_empty() && self.email.contains('@')
        }
    }
    
    #[test]
    fn test_user_creation() {
        let user = create_test_user();
        assert_eq!(user.id, 1);
        assert_eq!(user.name, "Test User");
        assert!(user.is_valid());
    }
    
    #[test]
    fn test_user_email_update() {
        let mut user = create_test_user();
        user.update_email("new@example.com");
        assert_eq!(user.email, "new@example.com");
    }

    // Pattern 2: Test with temp directory (teardown implicit via Drop)
    use std::path::PathBuf;
    use std::fs;
    
    struct TempDir {
        path: PathBuf,
    }
    
    impl TempDir {
        fn new(name: &str) -> Self {
            let path = std::env::temp_dir().join(name);
            fs::create_dir_all(&path).unwrap();
            TempDir { path }
        }
        
        fn path(&self) -> &PathBuf {
            &self.path
        }
    }
    
    impl Drop for TempDir {
        fn drop(&mut self) {
            if self.path.exists() {
                fs::remove_dir_all(&self.path).ok();
            }
        }
    }
    
    #[test]
    fn test_file_operations() {
        let temp = TempDir::new("test_file_ops_12345");
        let file_path = temp.path().join("test.txt");
        
        fs::write(&file_path, "Hello, Test!").unwrap();
        
        let content = fs::read_to_string(&file_path).unwrap();
        assert_eq!(content, "Hello, Test!");
        
        // temp goes out of scope here - directory cleaned up automatically
    }

    // Pattern 3: Shared state (once initialization)
    static INIT: Once = Once::new();
    static mut SHARED_DATA: Option<Vec<i32>> = None;
    
    fn get_shared_data() -> &'static Vec<i32> {
        unsafe {
            INIT.call_once(|| {
                SHARED_DATA = Some((1..=1000).collect());
            });
            SHARED_DATA.as_ref().unwrap()
        }
    }
    
    #[test]
    fn test_with_shared_data() {
        let data = get_shared_data();
        assert_eq!(data.len(), 1000);
        assert_eq!(data[0], 1);
        assert_eq!(data[999], 1000);
    }
    
    // Pattern 4: Builder pattern for test data
    struct UserBuilder {
        id: u32,
        name: String,
        email: String,
    }
    
    impl UserBuilder {
        fn new() -> Self {
            UserBuilder {
                id: 0,
                name: "Default".to_string(),
                email: "default@test.com".to_string(),
            }
        }
        
        fn id(mut self, id: u32) -> Self {
            self.id = id;
            self
        }
        
        fn name(mut self, name: &str) -> Self {
            self.name = name.to_string();
            self
        }
        
        fn email(mut self, email: &str) -> Self {
            self.email = email.to_string();
            self
        }
        
        fn build(self) -> TestUser {
            TestUser {
                id: self.id,
                name: self.name,
                email: self.email,
            }
        }
    }
    
    #[test]
    fn test_with_builder() {
        let user = UserBuilder::new()
            .id(42)
            .name("Alice")
            .email("alice@example.com")
            .build();
        
        assert_eq!(user.id, 42);
        assert_eq!(user.name, "Alice");
        assert!(user.is_valid());
    }
}
```

---

## 8. Mocking กับ Mockall

```toml
# Cargo.toml
[dev-dependencies]
mockall = "0.12"
```

```rust
use mockall::{automock, predicate::*};

// Define a trait that we want to mock
#[automock]
pub trait Database {
    fn find_user(&self, id: u32) -> Option<User>;
    fn save_user(&mut self, user: &User) -> bool;
    fn delete_user(&mut self, id: u32) -> bool;
    fn count_users(&self) -> usize;
}

#[derive(Debug, Clone, PartialEq)]
pub struct User {
    pub id: u32,
    pub name: String,
    pub email: String,
}

// Service that depends on Database trait
pub struct UserService<D: Database> {
    db: D,
}

impl<D: Database> UserService<D> {
    pub fn new(db: D) -> Self {
        UserService { db }
    }
    
    pub fn get_user(&self, id: u32) -> Option<User> {
        self.db.find_user(id)
    }
    
    pub fn create_user(&mut self, name: &str, email: &str) -> Result<User, String> {
        let id = self.db.count_users() as u32 + 1;
        let user = User {
            id,
            name: name.to_string(),
            email: email.to_string(),
        };
        
        if self.db.save_user(&user) {
            Ok(user)
        } else {
            Err("Failed to save user".to_string())
        }
    }
    
    pub fn remove_user(&mut self, id: u32) -> bool {
        self.db.delete_user(id)
    }
    
    pub fn update_email(&mut self, id: u32, new_email: &str) -> Result<(), String> {
        let mut user = self.db.find_user(id)
            .ok_or_else(|| format!("User {} not found", id))?;
        
        user.email = new_email.to_string();
        
        if self.db.save_user(&user) {
            Ok(())
        } else {
            Err("Failed to update user".to_string())
        }
    }
}

#[cfg(test)]
mod mock_tests {
    use super::*;

    #[test]
    fn test_get_user_found() {
        let mut mock_db = MockDatabase::new();
        
        // Setup expectation
        mock_db.expect_find_user()
            .with(eq(1))
            .times(1)
            .returning(|_| Some(User {
                id: 1,
                name: "Alice".to_string(),
                email: "alice@example.com".to_string(),
            }));
        
        let service = UserService::new(mock_db);
        let user = service.get_user(1);
        
        assert!(user.is_some());
        assert_eq!(user.unwrap().name, "Alice");
    }
    
    #[test]
    fn test_get_user_not_found() {
        let mut mock_db = MockDatabase::new();
        
        mock_db.expect_find_user()
            .with(eq(999))
            .times(1)
            .returning(|_| None);
        
        let service = UserService::new(mock_db);
        let user = service.get_user(999);
        
        assert!(user.is_none());
    }
    
    #[test]
    fn test_create_user_success() {
        let mut mock_db = MockDatabase::new();
        
        mock_db.expect_count_users()
            .times(1)
            .returning(|| 0);
        
        mock_db.expect_save_user()
            .times(1)
            .returning(|_| true);
        
        let mut service = UserService::new(mock_db);
        let result = service.create_user("Bob", "bob@example.com");
        
        assert!(result.is_ok());
        let user = result.unwrap();
        assert_eq!(user.id, 1);
        assert_eq!(user.name, "Bob");
    }
    
    #[test]
    fn test_create_user_failure() {
        let mut mock_db = MockDatabase::new();
        
        mock_db.expect_count_users()
            .times(1)
            .returning(|| 5);
        
        mock_db.expect_save_user()
            .times(1)
            .returning(|_| false);  // Simulate DB failure
        
        let mut service = UserService::new(mock_db);
        let result = service.create_user("Charlie", "charlie@example.com");
        
        assert!(result.is_err());
    }
    
    #[test]
    fn test_update_email() {
        let mut mock_db = MockDatabase::new();
        
        let original_user = User {
            id: 1,
            name: "Alice".to_string(),
            email: "alice@old.com".to_string(),
        };
        
        mock_db.expect_find_user()
            .with(eq(1))
            .times(1)
            .returning(move |_| Some(original_user.clone()));
        
        mock_db.expect_save_user()
            .times(1)
            .withf(|user| user.email == "alice@new.com")
            .returning(|_| true);
        
        let mut service = UserService::new(mock_db);
        let result = service.update_email(1, "alice@new.com");
        
        assert!(result.is_ok());
    }
}
```

---

## 9. Property-based Testing กับ Proptest

```toml
# Cargo.toml
[dev-dependencies]
proptest = "1.0"
```

```rust
pub fn sort_and_dedup(mut v: Vec<i32>) -> Vec<i32> {
    v.sort();
    v.dedup();
    v
}

pub fn reverse_string(s: &str) -> String {
    s.chars().rev().collect()
}

pub fn clamp(value: i32, min: i32, max: i32) -> i32 {
    if value < min { min }
    else if value > max { max }
    else { value }
}

pub fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 { let t = b; b = a % b; a = t; }
    a
}

#[cfg(test)]
mod property_tests {
    use super::*;
    use proptest::prelude::*;

    // proptest! macro ใช้ define property-based tests
    proptest! {
        // ทดสอบว่า sort_and_dedup ให้ผลลัพธ์ที่ถูกต้อง
        #[test]
        fn test_sort_sorted(v in proptest::collection::vec(any::<i32>(), 0..100)) {
            let result = sort_and_dedup(v);
            // Property 1: ผลลัพธ์ต้องเรียงแล้ว
            for w in result.windows(2) {
                prop_assert!(w[0] <= w[1], "Not sorted: {:?}", w);
            }
        }
        
        #[test]
        fn test_sort_no_duplicates(v in proptest::collection::vec(any::<i32>(), 0..100)) {
            let result = sort_and_dedup(v);
            // Property 2: ไม่มี duplicate
            for w in result.windows(2) {
                prop_assert!(w[0] != w[1], "Duplicate found: {:?}", w);
            }
        }

        // ทดสอบว่า reverse_string ทำงานถูกต้อง
        #[test]
        fn test_reverse_double_reverse(s in ".*") {
            // Property: reverse twice = original
            let reversed = reverse_string(&s);
            let double_reversed = reverse_string(&reversed);
            prop_assert_eq!(s, double_reversed);
        }
        
        #[test]
        fn test_reverse_length(s in ".*") {
            // Property: length ไม่เปลี่ยน
            let reversed = reverse_string(&s);
            prop_assert_eq!(s.chars().count(), reversed.chars().count());
        }

        // ทดสอบ clamp
        #[test]
        fn test_clamp_within_bounds(
            value in any::<i32>(),
            min in -1000i32..0,
            max in 0i32..1000
        ) {
            let result = clamp(value, min, max);
            // Property: result ต้องอยู่ใน [min, max]
            prop_assert!(result >= min, "result {} < min {}", result, min);
            prop_assert!(result <= max, "result {} > max {}", result, max);
        }
        
        #[test]
        fn test_clamp_identity_for_in_range(
            min in -1000i32..0,
            max in 0i32..1000,
            value_offset in 0i32..1000
        ) {
            // value is guaranteed to be in [min, max]
            let range = max - min;
            if range > 0 {
                let value = min + (value_offset % range);
                let result = clamp(value, min, max);
                prop_assert_eq!(value, result);
            }
        }

        // ทดสอบ GCD properties
        #[test]
        fn test_gcd_commutative(a in 1u64..10000, b in 1u64..10000) {
            // Property: gcd(a, b) = gcd(b, a)
            prop_assert_eq!(gcd(a, b), gcd(b, a));
        }
        
        #[test]
        fn test_gcd_divides_both(a in 1u64..10000, b in 1u64..10000) {
            let g = gcd(a, b);
            // Property: g หาร a และ b ลงตัว
            prop_assert_eq!(a % g, 0, "gcd {} does not divide {}", g, a);
            prop_assert_eq!(b % g, 0, "gcd {} does not divide {}", g, b);
        }
        
        #[test]
        fn test_gcd_identity(a in 1u64..10000) {
            // Property: gcd(a, a) = a
            prop_assert_eq!(gcd(a, a), a);
        }
    }
}
```

---

## 10. Benchmark Tests กับ Criterion

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_benchmarks"
harness = false
```

```rust
// benches/my_benchmarks.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

fn fibonacci_recursive(n: u64) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci_recursive(n - 1) + fibonacci_recursive(n - 2),
    }
}

fn fibonacci_iterative(n: u64) -> u64 {
    if n <= 1 { return n; }
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 2..=n {
        let c = a + b;
        a = b;
        b = c;
    }
    b
}

fn fibonacci_memo(n: u64) -> u64 {
    let mut memo = vec![0u64; (n + 1) as usize];
    memo[1] = 1;
    for i in 2..=n as usize {
        memo[i] = memo[i-1] + memo[i-2];
    }
    memo[n as usize]
}

// Sort benchmark
fn bubble_sort(v: &mut Vec<i32>) {
    let n = v.len();
    for i in 0..n {
        for j in 0..n - i - 1 {
            if v[j] > v[j + 1] {
                v.swap(j, j + 1);
            }
        }
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    let mut group = c.benchmark_group("fibonacci");
    
    for n in [10u64, 20, 30].iter() {
        group.bench_with_input(
            BenchmarkId::new("recursive", n),
            n,
            |b, &n| b.iter(|| fibonacci_recursive(black_box(n)))
        );
        
        group.bench_with_input(
            BenchmarkId::new("iterative", n),
            n,
            |b, &n| b.iter(|| fibonacci_iterative(black_box(n)))
        );
        
        group.bench_with_input(
            BenchmarkId::new("memoized", n),
            n,
            |b, &n| b.iter(|| fibonacci_memo(black_box(n)))
        );
    }
    
    group.finish();
}

fn bench_sort(c: &mut Criterion) {
    let mut group = c.benchmark_group("sorting");
    
    for size in [100usize, 1000, 10000].iter() {
        let data: Vec<i32> = (0..*size as i32).rev().collect();
        
        group.bench_with_input(
            BenchmarkId::new("bubble_sort", size),
            size,
            |b, _| {
                b.iter(|| {
                    let mut v = data.clone();
                    bubble_sort(&mut v);
                    v
                })
            }
        );
        
        group.bench_with_input(
            BenchmarkId::new("std_sort", size),
            size,
            |b, _| {
                b.iter(|| {
                    let mut v = data.clone();
                    v.sort();
                    v
                })
            }
        );
    }
    
    group.finish();
}

fn bench_string_operations(c: &mut Criterion) {
    c.bench_function("string_concat_push", |b| {
        b.iter(|| {
            let mut s = String::new();
            for i in 0..100 {
                s.push_str(&i.to_string());
            }
            black_box(s)
        })
    });
    
    c.bench_function("string_concat_collect", |b| {
        b.iter(|| {
            let s: String = (0..100)
                .map(|i| i.to_string())
                .collect();
            black_box(s)
        })
    });
    
    c.bench_function("string_concat_join", |b| {
        b.iter(|| {
            let parts: Vec<String> = (0..100).map(|i| i.to_string()).collect();
            let s = parts.join("");
            black_box(s)
        })
    });
}

criterion_group!(benches, bench_fibonacci, bench_sort, bench_string_operations);
criterion_main!(benches);
```

---

## 11. Test Utilities และ Custom Assertions

```rust
// src/test_utils.rs (only compiled in test mode)
#[cfg(test)]
pub mod helpers {
    use std::fmt::Debug;
    
    /// Assert that two floating point numbers are approximately equal
    pub fn assert_approx_eq(a: f64, b: f64, epsilon: f64) {
        let diff = (a - b).abs();
        assert!(
            diff < epsilon,
            "Expected {} ≈ {} (epsilon: {}), actual diff: {}",
            a, b, epsilon, diff
        );
    }
    
    /// Assert that a Vec is sorted
    pub fn assert_sorted<T: Ord + Debug>(v: &[T]) {
        for i in 1..v.len() {
            assert!(
                v[i - 1] <= v[i],
                "Vec not sorted at index {}: {:?} > {:?}",
                i, v[i - 1], v[i]
            );
        }
    }
    
    /// Assert that two Vecs contain the same elements (regardless of order)
    pub fn assert_same_elements<T: Ord + Clone + Debug>(a: &[T], b: &[T]) {
        let mut a_sorted = a.to_vec();
        let mut b_sorted = b.to_vec();
        a_sorted.sort();
        b_sorted.sort();
        assert_eq!(a_sorted, b_sorted,
            "Vectors don't contain same elements:\n  left:  {:?}\n  right: {:?}", a, b);
    }
    
    /// Time a function and assert it completes within a budget
    pub fn assert_time_budget_ms<F, R>(f: F, budget_ms: u64) -> R
    where F: FnOnce() -> R
    {
        let start = std::time::Instant::now();
        let result = f();
        let elapsed = start.elapsed().as_millis() as u64;
        assert!(
            elapsed <= budget_ms,
            "Function took {}ms, budget was {}ms",
            elapsed, budget_ms
        );
        result
    }
}

#[cfg(test)]
mod tests {
    use super::helpers::*;
    
    #[test]
    fn test_approx_eq() {
        assert_approx_eq(0.1 + 0.2, 0.3, 1e-10);
        assert_approx_eq(std::f64::consts::PI, 3.14159265, 1e-5);
    }
    
    #[test]
    fn test_sorted_check() {
        let sorted = vec![1, 2, 3, 4, 5];
        assert_sorted(&sorted);
        
        let mut data = vec![5, 3, 1, 4, 2];
        data.sort();
        assert_sorted(&data);
    }
    
    #[test]
    fn test_same_elements() {
        let a = vec![1, 2, 3, 4, 5];
        let b = vec![5, 3, 1, 4, 2];
        assert_same_elements(&a, &b);
    }
    
    #[test]
    fn test_time_budget() {
        let result = assert_time_budget_ms(
            || {
                // Fast computation
                (0..1000).sum::<i32>()
            },
            100  // 100ms budget
        );
        assert_eq!(result, 499500);
    }
}
```

---

## 12. Running Tests

```bash
# Run all tests
cargo test

# Run tests with output
cargo test -- --nocapture

# Run specific test by name
cargo test test_add

# Run tests in a specific module
cargo test calculator::tests

# Run only unit tests (no integration)
cargo test --lib

# Run only integration tests
cargo test --test integration_test

# Run benchmarks
cargo bench

# Run with multiple threads
cargo test -- --test-threads=4

# List all tests without running
cargo test -- --list

# Run ignored tests
cargo test -- --ignored

# Show test timing
cargo test -- --report-time
```

```rust
// Ignoring tests
#[cfg(test)]
mod ignore_tests {
    #[test]
    #[ignore]  // ข้ามใน default run
    fn expensive_test() {
        // Test ที่ใช้เวลานาน
        std::thread::sleep(std::time::Duration::from_secs(60));
    }
    
    #[test]
    #[ignore = "requires external database"]
    fn database_integration_test() {
        // Test ที่ต้องการ external resources
    }
    
    #[test]
    fn normal_test() {
        assert!(true);  // This runs normally
    }
}
```

---

## 13. สรุป

| Concept | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Unit test | `#[test]` | ทดสอบ function เดียว |
| Test module | `#[cfg(test)] mod tests` | รวม tests ใน file เดียวกัน |
| Should panic | `#[should_panic]` | ทดสอบ panic behavior |
| Result test | `-> Result<(), E>` | test ที่ return Result |
| Integration test | `tests/` directory | ทดสอบ public API |
| Ignored | `#[ignore]` | ข้าม expensive tests |
| Benchmark | Criterion | วัด performance |
| Property test | proptest | ทดสอบ properties |
| Mock | mockall | แทนที่ dependencies |

### Testing Philosophy

1. **Test พฤติกรรม ไม่ใช่ implementation** - ทดสอบสิ่งที่ function ควรทำ
2. **AAA Pattern** - Arrange, Act, Assert
3. **Fast tests** - unit tests ควรเร็วมาก (< 1ms)
4. **Independent tests** - แต่ละ test ต้องทำงานอิสระ
5. **Descriptive names** - ชื่อ test ควรบอกว่าทดสอบอะไร

---

*[← Part 012: Modules, Crates, และ Packages](../part_012/README.md) | [Part 014: File I/O และ Filesystem →](../part_014/README.md)*
