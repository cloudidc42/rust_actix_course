# Part 004: Ownership, Borrowing, และ References 🔑

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Ownership rules
- เข้าใจ Move semantics
- ใช้ References และ Borrowing ได้
- เข้าใจ Borrow Checker
- หลีกเลี่ยง common ownership errors

---

## 1. ทำไม Ownership ถึงสำคัญ?

```
ปัญหา Memory ในภาษาอื่น:

C/C++ (Manual Memory):
char* s1 = malloc(100);
char* s2 = s1;  // s1 และ s2 ชี้ไปที่เดียวกัน
free(s1);
printf("%s", s2);  // ❌ Use-after-free bug!

Java/Python (Garbage Collection):
Object obj = new Object();
// GC ดูแลเอง แต่:
// - Stop-the-world pauses
// - ไม่ predictable เมื่อ GC จะทำงาน
// - Runtime overhead

Rust (Ownership):
let s1 = String::from("hello");
let s2 = s1;  // s1 ถูก MOVE ไปที่ s2
// println!("{}", s1);  ← COMPILE ERROR! s1 ไม่มีแล้ว
println!("{}", s2);  // ✅ OK
// s2 จะถูกทำลายเมื่อออก scope โดยอัตโนมัติ
// ไม่มี GC, ไม่ leak, ปลอดภัย 100%
```

---

## 2. Ownership Rules

**กฎพื้นฐาน 3 ข้อ:**
1. แต่ละ value ใน Rust มี **owner** หนึ่งคน
2. ในเวลาหนึ่งๆ มี owner ได้เพียงคนเดียว
3. เมื่อ owner ออก scope, value จะถูกทำลาย (drop)

```rust
fn main() {
    // 1. x เป็น owner ของ value 5
    let x = 5;
    println!("x = {}", x);
    // เมื่อออก scope, x (และ value 5) จะถูก destroy

    // 2. s เป็น owner ของ String "hello"
    let s = String::from("hello");
    println!("s = {}", s);
    // เมื่อออก scope, s จะถูก destroy + memory ถูก free

    {
        // inner scope
        let inner = String::from("inner");
        println!("inner = {}", inner);
    }  // ← inner ถูก destroy ที่นี่
    // println!("{}", inner);  // ERROR! inner ไม่อยู่ใน scope แล้ว
}  // ← s ถูก destroy ที่นี่ (drop() ถูกเรียก)
```

---

## 3. Move Semantics

### 3.1 Move กับ Types ที่ Heap-allocated

```rust
fn main() {
    // String เก็บใน heap → Move
    let s1 = String::from("hello");
    let s2 = s1;  // s1 ถูก MOVE ไปที่ s2

    // println!("{}", s1);  // ERROR: value borrowed here after move
    println!("{}", s2);  // OK

    // ทำไม Rust ไม่ copy แบบ deep copy?
    // เพราะ String อาจใหญ่มาก deep copy จะช้า
    // Rust บังคับให้ programmer ตัดสินใจเอง

    // ถ้าต้องการ copy ต้อง clone() อย่างชัดเจน
    let s3 = String::from("world");
    let s4 = s3.clone();  // explicit deep copy

    println!("s3 = {}", s3);  // OK (s3 ยังมีอยู่)
    println!("s4 = {}", s4);  // OK
}
```

### 3.2 Copy Types (ไม่ Move)

```rust
fn main() {
    // Types ที่ implement Copy trait: integers, floats, bool, char, tuples ของ Copy types
    let x = 5;
    let y = x;  // COPY ไม่ใช่ move!
    println!("x = {}, y = {}", x, y);  // ทั้งคู่ใช้ได้

    // Float
    let a = 3.14;
    let b = a;
    println!("a = {}, b = {}", a, b);

    // Bool
    let flag = true;
    let flag2 = flag;
    println!("{} {}", flag, flag2);

    // Tuple ของ Copy types
    let t1 = (1, 2.0, true);
    let t2 = t1;
    println!("{:?} {:?}", t1, t2);

    // Array ของ Copy types
    let arr1 = [1, 2, 3];
    let arr2 = arr1;
    println!("{:?} {:?}", arr1, arr2);

    // String ไม่ใช่ Copy!
    let s1 = String::from("hello");
    // let s2 = s1;  // ถ้าทำแบบนี้ s1 จะหายไป
    let s2 = s1.clone();  // ต้อง clone
    println!("{} {}", s1, s2);
}
```

### 3.3 Move กับ Functions

```rust
fn takes_ownership(s: String) {
    println!("Got: {}", s);
    // s ถูก drop เมื่อออก function
}

fn makes_copy(x: i32) {
    println!("Got: {}", x);
    // x เป็น Copy ดังนั้น original ยังอยู่
}

fn gives_ownership() -> String {
    String::from("new string")  // return ค่า → transfer ownership ออกไป
}

fn takes_and_gives_back(s: String) -> String {
    s  // return s → ownership ย้ายออกไป
}

fn main() {
    let s1 = String::from("hello");
    takes_ownership(s1);        // s1 ถูก move เข้า function
    // println!("{}", s1);      // ERROR! s1 ไม่มีแล้ว

    let x = 5;
    makes_copy(x);              // x เป็น copy
    println!("x = {}", x);     // OK! x ยังมีอยู่

    let s2 = gives_ownership();    // function สร้าง string ใหม่
    println!("s2 = {}", s2);

    let s3 = String::from("world");
    let s4 = takes_and_gives_back(s3);  // s3 move เข้า, ownership return กลับมาเป็น s4
    // println!("{}", s3);             // ERROR! s3 ไม่มีแล้ว
    println!("s4 = {}", s4);
}
```

---

## 4. References และ Borrowing

### 4.1 Immutable References

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
    // s เป็นแค่ reference ไม่ได้ own ดังนั้นไม่ถูก drop
}

fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

fn main() {
    let s1 = String::from("hello world");

    // & = borrow (สร้าง reference)
    let len = calculate_length(&s1);
    println!("'{}' has {} characters", s1, len);  // s1 ยังใช้ได้!

    // Multiple immutable references = OK
    let r1 = &s1;
    let r2 = &s1;
    let r3 = &s1;
    println!("{} {} {}", r1, r2, r3);  // ทั้ง 3 ใช้งานพร้อมกันได้

    // String slice
    let word = first_word(&s1);
    println!("First word: '{}'", word);

    // &str vs &String
    let s = String::from("hello");
    let slice1: &str = &s;        // String → &str
    let slice2: &str = "literal"; // string literal เป็น &str อยู่แล้ว

    println!("{} {}", slice1, slice2);
}
```

### 4.2 Mutable References

```rust
fn change(s: &mut String) {
    s.push_str(", world");
}

fn append_number(v: &mut Vec<i32>, n: i32) {
    v.push(n);
}

fn main() {
    // ต้องประกาศ mut และส่ง &mut
    let mut s = String::from("hello");
    change(&mut s);
    println!("{}", s);  // "hello, world"

    let mut numbers = vec![1, 2, 3];
    append_number(&mut numbers, 4);
    append_number(&mut numbers, 5);
    println!("{:?}", numbers);  // [1, 2, 3, 4, 5]

    // RULE: ในเวลาหนึ่งๆ มี mutable reference ได้เพียง 1 อัน
    let mut data = String::from("hello");
    let r1 = &mut data;
    // let r2 = &mut data;  // ERROR! ไม่สามารถมี 2 mutable references พร้อมกัน
    println!("{}", r1);

    // หลังจาก r1 ถูกใช้งานครั้งสุดท้ายแล้ว ค่อยสร้าง r2 ได้
    let r1 = &mut data;
    r1.push_str(" rust");
    // r1 ไม่ถูกใช้หลังจากนี้แล้ว (NLL - Non-Lexical Lifetimes)

    let r2 = &mut data;
    r2.push_str("!");
    println!("{}", r2);  // "hello rust!"
}
```

### 4.3 Borrow Checker Rules

```rust
fn main() {
    // RULE 1: ไม่สามารถมี mutable reference พร้อมกับ immutable reference
    let mut s = String::from("hello");

    let r1 = &s;     // immutable borrow
    let r2 = &s;     // another immutable borrow - OK
    // let r3 = &mut s;  // ERROR! มี immutable borrows อยู่แล้ว
    println!("{} {}", r1, r2);  // r1, r2 ใช้งานสุดท้ายที่นี่

    // หลังจาก r1, r2 ไม่ถูกใช้แล้ว สามารถ borrow mut ได้
    let r3 = &mut s;
    r3.push_str(", world");
    println!("{}", r3);

    // RULE 2: Dangling references ไม่สามารถเกิดได้
    // fn dangling() -> &String {
    //     let s = String::from("hello");
    //     &s  // ERROR! s จะถูก drop เมื่อออก function
    //         // &s จะกลายเป็น dangling reference
    // }

    // CORRECT: return owned value แทน
    fn no_dangling() -> String {
        let s = String::from("hello");
        s  // move ownership ออกไป
    }
    let s = no_dangling();
    println!("{}", s);
}
```

---

## 5. Slice Types

### 5.1 String Slices

```rust
fn main() {
    let s = String::from("hello world");

    // String slices
    let hello = &s[0..5];    // "hello"
    let world = &s[6..11];   // "world"
    let full  = &s[..];      // "hello world"
    let start = &s[..5];     // "hello"
    let end   = &s[6..];     // "world"

    println!("{} {} {}", hello, world, full);

    // String literals เป็น slice อยู่แล้ว
    let s_literal = "hello";  // type: &str (ชี้ไปที่ binary)

    // Function รับ &str ดีกว่า &String (flexible กว่า)
    fn process(s: &str) -> usize {
        s.len()
    }

    let owned = String::from("owned string");
    let literal = "literal string";

    println!("{}", process(&owned));    // &String → &str (coercion)
    println!("{}", process(literal));   // &str ตรงๆ

    // Multibyte characters (Thai, emoji)
    let thai = "สวัสดี";
    println!("chars: {}", thai.chars().count());  // 6 characters
    println!("bytes: {}", thai.len());            // 18 bytes (UTF-8)

    // ระวัง! slice ต้องตรงขอบ character
    // let bad = &thai[0..3];  // อาจ panic ถ้าตัดกลาง character
    let good = &thai[0..3];   // "ส" (3 bytes ใน UTF-8)
    println!("first char: {}", good);

    // ปลอดภัยกว่า ใช้ chars()
    let first_char: String = thai.chars().take(1).collect();
    println!("first char (safe): {}", first_char);
}
```

### 5.2 Array Slices

```rust
fn sum_slice(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}

fn find_in_slice(data: &[i32], target: i32) -> Option<usize> {
    data.iter().position(|&x| x == target)
}

fn main() {
    let arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // Slices ของ array
    let first3 = &arr[..3];
    let last3  = &arr[7..];
    let middle = &arr[3..7];

    println!("first3: {:?}", first3);  // [1, 2, 3]
    println!("last3: {:?}", last3);    // [8, 9, 10]
    println!("middle: {:?}", middle);  // [4, 5, 6, 7]

    println!("sum of all: {}", sum_slice(&arr));
    println!("sum of first3: {}", sum_slice(first3));

    match find_in_slice(&arr, 7) {
        Some(i) => println!("Found 7 at index {}", i),
        None => println!("Not found"),
    }

    // Mutable slice
    let mut data = [3, 1, 4, 1, 5, 9, 2, 6];
    let slice = &mut data[2..6];
    slice.sort();
    println!("After sorting middle: {:?}", data);

    // Vec slices
    let v = vec![10, 20, 30, 40, 50];
    let vs = &v[1..4];
    println!("vec slice: {:?}", vs);

    // Slice patterns
    let numbers = [1, 2, 3, 4, 5];
    match numbers {
        [first, .., last] => println!("first={}, last={}", first, last),
    }

    match numbers.as_ref() {
        [a, b, rest @ ..] => {
            println!("a={}, b={}", a, b);
            println!("rest={:?}", rest);
        }
    }
}
```

---

## 6. ตัวอย่างจริง: String Processing

```rust
// Text processing functions ที่ใช้ references อย่างถูกต้อง

fn word_count(text: &str) -> usize {
    text.split_whitespace().count()
}

fn word_frequency(text: &str) -> std::collections::HashMap<&str, usize> {
    let mut freq = std::collections::HashMap::new();
    for word in text.split_whitespace() {
        *freq.entry(word).or_insert(0) += 1;
    }
    freq
}

fn longest_word<'a>(text: &'a str) -> &'a str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

fn reverse_words(text: &str) -> String {
    text.split_whitespace()
        .rev()
        .collect::<Vec<&str>>()
        .join(" ")
}

fn capitalize_words(text: &str) -> String {
    text.split_whitespace()
        .map(|word| {
            let mut chars = word.chars();
            match chars.next() {
                None => String::new(),
                Some(first) => {
                    first.to_uppercase().to_string() + chars.as_str()
                }
            }
        })
        .collect::<Vec<String>>()
        .join(" ")
}

fn main() {
    let text = "the quick brown fox jumps over the lazy dog";

    println!("Text: \"{}\"", text);
    println!("Words: {}", word_count(text));
    println!("Longest word: '{}'", longest_word(text));
    println!("Reversed: '{}'", reverse_words(text));
    println!("Capitalized: '{}'", capitalize_words(text));

    println!("\nWord frequencies:");
    let mut freq: Vec<_> = word_frequency(text).into_iter().collect();
    freq.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    for (word, count) in &freq {
        if *count > 1 {
            println!("  '{}': {} times", word, count);
        }
    }
}
```

---

## 7. สรุปและ Exercises

### 7.1 สิ่งที่เรียนรู้

✅ Ownership rules (owner เดียว, drop เมื่อออก scope)  
✅ Move semantics สำหรับ heap types  
✅ Copy types (integers, booleans, etc.)  
✅ Immutable references (&T)  
✅ Mutable references (&mut T)  
✅ Borrow Checker rules  
✅ String slices (&str)  
✅ Array/Vec slices  

### 7.2 Exercises

**Exercise 1: String Functions**
```rust
// เขียน functions ต่อไปนี้โดยใช้ references อย่างถูกต้อง:
// fn is_palindrome(s: &str) -> bool
// fn count_vowels(s: &str) -> usize
// fn remove_duplicates(s: &str) -> String
```

**Exercise 2: Number Processing**
```rust
// fn largest_in_slice(numbers: &[i32]) -> i32
// fn second_largest(numbers: &[i32]) -> Option<i32>
// fn rotate_slice(numbers: &mut Vec<i32>, k: usize)
```

**Exercise 3: Understanding Errors**
```rust
// แก้ไข code ต่อไปนี้ให้ compile ได้:

fn main() {
    let s = String::from("hello");
    let r1 = &s;
    let r2 = &mut s;  // แก้ตรงนี้
    println!("{} {}", r1, r2);
}
```

---

*[← Part 003: Functions และ Control Flow](../part_003/README.md) | [Part 005: Structs และ Enums →](../part_005/README.md)*
