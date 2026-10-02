# Part 010: Lifetimes และ Memory Safety ⏳

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Lifetimes อย่างลึกซึ้ง
- Lifetime annotations
- Lifetime elision rules
- Static lifetimes
- Advanced lifetime scenarios

---

## 1. ทำไมต้องมี Lifetimes?

```rust
// ปัญหา: function return reference - Rust ไม่รู้ว่า reference มาจากไหน
// fn longest(x: &str, y: &str) -> &str {  // ERROR: missing lifetime specifier
//     if x.len() > y.len() { x } else { y }
// }

// Lifetime annotation: บอก Rust ว่า return value มี lifetime เท่ากับ input ที่สั้นกว่า
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("Longest: {}", result);  // OK: result ใช้ใน scope ของ string2
    }
    // println!("{}", result);  // ERROR: string2 ถูก drop แล้ว
}
```

---

## 2. Lifetime Annotations

### 2.1 Function Lifetimes

```rust
// 'a บอกว่า return value มีชีวิตอย่างน้อยเท่ากับ x และ y
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

// Return value มาจาก x เสมอ → lifetime ต้องเหมือน x เท่านั้น
fn first_word<'a>(s: &'a str) -> &'a str {
    s.split_whitespace().next().unwrap_or(s)
}

// y ไม่เกี่ยวกับ return value
fn always_returns_x<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    println!("y={}", y);  // use y to avoid warning
    x
}

fn main() {
    let s1 = String::from("hello world");
    let s2 = String::from("hi");

    let longest = longest(s1.as_str(), s2.as_str());
    println!("Longest: {}", longest);

    let word = first_word(&s1);
    println!("First word: {}", word);
}
```

### 2.2 Struct Lifetimes

```rust
// Struct ที่เก็บ reference ต้องมี lifetime annotation
#[derive(Debug)]
struct Important<'a> {
    part: &'a str,
}

impl<'a> Important<'a> {
    fn announce(&self, announcement: &str) -> &str {
        println!("Attention! {}", announcement);
        self.part
    }

    // Return value lifetime = self lifetime
    fn level(&self) -> &str {
        self.part
    }
}

// ตัวอย่างจริง: Parser ที่ borrow จาก input string
#[derive(Debug)]
struct Parser<'a> {
    input: &'a str,
    position: usize,
}

impl<'a> Parser<'a> {
    fn new(input: &'a str) -> Self {
        Self { input, position: 0 }
    }

    fn peek(&self) -> Option<char> {
        self.input[self.position..].chars().next()
    }

    fn advance(&mut self) -> Option<char> {
        let c = self.peek()?;
        self.position += c.len_utf8();
        Some(c)
    }

    fn parse_word(&mut self) -> &'a str {
        let start = self.position;
        while let Some(c) = self.peek() {
            if c.is_alphabetic() {
                self.position += c.len_utf8();
            } else {
                break;
            }
        }
        &self.input[start..self.position]
    }

    fn skip_whitespace(&mut self) {
        while let Some(c) = self.peek() {
            if c.is_whitespace() {
                self.position += c.len_utf8();
            } else {
                break;
            }
        }
    }

    fn parse_all_words(&mut self) -> Vec<&'a str> {
        let mut words = Vec::new();
        loop {
            self.skip_whitespace();
            if self.position >= self.input.len() {
                break;
            }
            let word = self.parse_word();
            if !word.is_empty() {
                words.push(word);
            }
        }
        words
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence;
    {
        let i = novel.find('.').unwrap_or(novel.len());
        first_sentence = &novel[..i];
    }

    let important = Important { part: first_sentence };
    println!("{:?}", important);
    println!("{}", important.level());

    // Parser example
    let text = "hello world rust programming";
    let mut parser = Parser::new(text);
    let words = parser.parse_all_words();
    println!("Words: {:?}", words);

    // Words borrow from text, so text must outlive words
    for word in &words {
        println!("  '{}'", word);
    }
}
```

---

## 3. Lifetime Elision Rules

```rust
// Rust มี 3 กฎสำหรับ elide lifetimes (ไม่ต้องเขียน)

// Rule 1: แต่ละ input reference ได้ lifetime ของตัวเอง
// fn foo(x: &str)              → fn foo<'a>(x: &'a str)
// fn foo(x: &str, y: &str)     → fn foo<'a, 'b>(x: &'a str, y: &'b str)

// Rule 2: ถ้ามี input lifetime เดียว → output ได้ lifetime นั้น
// fn foo(x: &str) -> &str      → fn foo<'a>(x: &'a str) -> &'a str

// Rule 3: ถ้ามี &self หรือ &mut self → output ได้ lifetime ของ self
// impl Foo { fn bar(&self) -> &str  → fn bar<'a>(&'a self) -> &'a str }

// ตัวอย่าง: ไม่ต้องเขียน lifetime annotations เหล่านี้
fn first(s: &str) -> &str {  // rule 1 + rule 2
    s.split_whitespace().next().unwrap_or(s)
}

struct Counter {
    data: Vec<String>,
}

impl Counter {
    fn get(&self, index: usize) -> Option<&str> {  // rule 3
        self.data.get(index).map(|s| s.as_str())
    }
}

// แต่ต้องเขียน lifetime เมื่อ compiler ไม่รู้
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}

fn main() {
    let s = "hello world";
    println!("{}", first(s));

    let counter = Counter {
        data: vec!["one".to_string(), "two".to_string()],
    };
    println!("{:?}", counter.get(0));
}
```

---

## 4. Static Lifetime

```rust
// 'static: มีชีวิตตลอดโปรแกรม
fn main() {
    // String literals มี 'static lifetime
    let s: &'static str = "I have a static lifetime";
    println!("{}", s);

    // Constants มี 'static lifetime
    const HELLO: &str = "Hello, World!";
    println!("{}", HELLO);

    // Static references
    static GLOBAL: &str = "I am global";
    println!("{}", GLOBAL);

    // Thread spawn ต้องการ 'static (เพราะ thread อาจ outlive scope)
    let s = String::from("hello");
    let handle = std::thread::spawn(move || {  // move ไม่ใช่ 'static reference
        println!("In thread: {}", s);
    });
    handle.join().unwrap();

    // เมื่อใดจึงใช้ 'static:
    // 1. String literals
    // 2. Global data
    // 3. Error messages ที่ hardcoded
    // 4. Types ที่ต้องส่งผ่าน threads

    // generic bound 'static
    fn must_be_static<T: 'static>(value: T) -> T {
        value
    }

    let owned = String::from("owned");  // String ไม่มี references → 'static
    let result = must_be_static(owned);
    println!("{}", result);
}
```

---

## 5. Advanced: Lifetime Subtyping

```rust
// 'a: 'b แปลว่า 'a อย่างน้อยนานเท่า 'b
fn longest_with_announcement<'a, 'b>(
    x: &'a str,
    y: &'a str,
    ann: &'b str,
) -> &'a str
where 'b: 'a  // 'b ต้องนานอย่างน้อยเท่า 'a
{
    println!("Announcement: {}", ann);
    if x.len() > y.len() { x } else { y }
}

// Higher-ranked trait bounds (HRTB)
fn apply_twice<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(f(x))
}

fn main() {
    let s1 = String::from("long string");
    let s2 = String::from("xy");
    let ann = String::from("Important announcement");

    let result = longest_with_announcement(
        s1.as_str(),
        s2.as_str(),
        ann.as_str(),
    );
    println!("Longest: {}", result);

    let double = |x| x * 2;
    println!("double twice: {}", apply_twice(double, 3));  // 12
}
```

---

## 6. สรุปและ Exercises

### 6.1 สิ่งที่เรียนรู้

✅ ทำไมต้องมี Lifetime annotations  
✅ Lifetime syntax `'a`  
✅ Lifetimes ใน functions  
✅ Lifetimes ใน structs  
✅ Lifetime elision rules  
✅ Static lifetime  

### 6.2 Exercise

**Exercise: String Cache**
```rust
struct StringCache<'a> {
    strings: Vec<&'a str>,
}

impl<'a> StringCache<'a> {
    fn add(&mut self, s: &'a str) { ... }
    fn find(&self, prefix: &str) -> Vec<&'a str> { ... }
    fn longest(&self) -> Option<&'a str> { ... }
}
```

---

*[← Part 009: Traits และ Generics](../part_009/README.md) | [Part 011: Closures และ Iterators →](../part_011/README.md)*
