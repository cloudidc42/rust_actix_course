# Part 002: ตัวแปร, ชนิดข้อมูล, และ Mutability 🔢

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- ประกาศตัวแปรแบบต่างๆ ได้
- เข้าใจ Mutability และ Immutability
- รู้จักชนิดข้อมูลพื้นฐานทั้งหมดใน Rust
- เข้าใจ Type Inference
- ใช้ Constants และ Static Variables ได้
- เข้าใจ Shadowing

---

## 1. Variables และ Mutability

### 1.1 Immutable Variables (Default)

ใน Rust ตัวแปรทุกตัว **immutable โดย default** นี่คือความแตกต่างจากภาษาอื่น:

```rust
fn main() {
    // ตัวแปร immutable (ไม่สามารถเปลี่ยนค่าได้)
    let x = 5;
    println!("x = {}", x);

    // ERROR: cannot assign twice to immutable variable
    // x = 6;  // ← จะ compile error!

    // ต้องใช้ mut keyword เพื่อให้เปลี่ยนค่าได้
    let mut y = 5;
    println!("y = {}", y);
    y = 6;  // OK!
    println!("y = {}", y);
}
```

### 1.2 ทำไม Rust ถึงทำแบบนี้?

```
❌ Bug ที่พบบ่อยในภาษาอื่น:
x = 5
// ...100 บรรทัดต่อมา...
x = "hello"  // ลืมว่า x เคยเป็นตัวเลข

✅ Rust ช่วยป้องกัน:
let x = 5;
x = "hello";  // COMPILE ERROR! ชัดเจนทันที

ประโยชน์:
1. Code อ่านง่ายขึ้น - รู้ว่าค่าไม่เปลี่ยน
2. Thread safety - immutable values ปลอดภัยระหว่าง threads
3. Optimization - compiler optimize ได้ดีกว่า
```

### 1.3 Mutable Variables

```rust
fn main() {
    let mut count = 0;
    println!("count = {}", count);  // 0

    count += 1;
    println!("count = {}", count);  // 1

    count += 1;
    println!("count = {}", count);  // 2

    // เปลี่ยนค่าได้ แต่ต้องเป็นชนิดเดิม
    let mut name = String::from("สมชาย");
    println!("name = {}", name);

    name = String::from("สมหญิง");
    println!("name = {}", name);

    // ERROR: expected `String`, found `i32`
    // name = 42;
}
```

---

## 2. Constants

Constants ต่างจาก variables ตรงที่:
- ใช้ `const` แทน `let`
- **ต้องระบุชนิดข้อมูลเสมอ**
- ค่าต้องเป็น constant expression (ไม่ใช่ runtime value)
- convention: ตัวพิมพ์ใหญ่ทั้งหมด + underscore

```rust
// Constants (global scope)
const MAX_POINTS: u32 = 100_000;
const PI: f64 = 3.14159265358979;
const APP_NAME: &str = "MyRustApp";
const SECONDS_PER_HOUR: u32 = 60 * 60;  // expressions ได้

fn main() {
    println!("Max points: {}", MAX_POINTS);
    println!("Pi: {}", PI);
    println!("App: {}", APP_NAME);
    println!("Seconds/hour: {}", SECONDS_PER_HOUR);

    // Constants ใช้ได้ใน block ด้วย
    const LOCAL_MAX: i32 = 50;
    println!("Local max: {}", LOCAL_MAX);

    // สูตรคำนวณ
    let area = PI * 5.0 * 5.0;
    println!("Area of circle with r=5: {:.2}", area);
}
```

---

## 3. Static Variables

Static variables อยู่ตลอด lifetime ของโปรแกรม:

```rust
// Static immutable
static GREETING: &str = "สวัสดี";
static VERSION: u32 = 1;

// Static mutable (ต้องระวัง - unsafe)
static mut COUNTER: u32 = 0;

fn main() {
    println!("{} จากเวอร์ชัน {}", GREETING, VERSION);

    // การเข้าถึง static mut ต้องอยู่ใน unsafe block
    unsafe {
        COUNTER += 1;
        println!("Counter: {}", COUNTER);
    }

    // Static string slice
    let s: &'static str = "นี่คือ static string";
    println!("{}", s);
}
```

---

## 4. Shadowing

Shadowing คือการประกาศตัวแปรชื่อเดิมซ้ำ ซึ่งจะ "บัง" ตัวแปรเดิม:

```rust
fn main() {
    // Shadowing พื้นฐาน
    let x = 5;
    println!("x = {}", x);  // 5

    let x = x + 1;  // shadow x ใหม่
    println!("x = {}", x);  // 6

    {
        let x = x * 2;  // shadow ใน inner scope
        println!("inner x = {}", x);  // 12
    }

    println!("x = {}", x);  // 6 (กลับมาใช้ outer x)

    // -------------------------------------------
    // ข้อดีของ Shadowing: เปลี่ยนชนิดได้!
    // -------------------------------------------

    let spaces = "   ";  // &str
    println!("spaces = '{}'", spaces);

    let spaces = spaces.len();  // usize (shadow ด้วยชนิดใหม่)
    println!("spaces length = {}", spaces);

    // mut ทำไม่ได้แบบนี้:
    // let mut spaces = "   ";
    // spaces = spaces.len();  // ERROR: type mismatch

    // -------------------------------------------
    // Shadowing ใช้งานจริง
    // -------------------------------------------

    // แปลง input string เป็นตัวเลข
    let input = "42";
    let input: i32 = input.parse().expect("ไม่ใช่ตัวเลข");
    println!("input as number = {}", input);

    // แปลง
    let temperature = 100.0_f64;  // Celsius
    let temperature = temperature * 9.0 / 5.0 + 32.0;  // Fahrenheit
    println!("Temperature: {:.1}°F", temperature);
}
```

---

## 5. ชนิดข้อมูลพื้นฐาน (Primitive Types)

### 5.1 Integer Types

```rust
fn main() {
    // ---------------
    // Signed Integers
    // ---------------
    let a: i8  = 127;        // -128 ถึง 127
    let b: i16 = 32_767;     // -32,768 ถึง 32,767
    let c: i32 = 2_147_483_647; // ±2.1 billion
    let d: i64 = 9_223_372_036_854_775_807; // ±9.2 quintillion
    let e: i128 = 170_141_183_460_469_231_731_687_303_715_884_105_727; // ใหญ่มาก
    let f: isize = 100;      // ขึ้นกับ architecture (32 หรือ 64 bit)

    // -----------------
    // Unsigned Integers
    // -----------------
    let g: u8  = 255;         // 0 ถึง 255
    let h: u16 = 65_535;      // 0 ถึง 65,535
    let i: u32 = 4_294_967_295; // 0 ถึง 4.3 billion
    let j: u64 = 18_446_744_073_709_551_615; // 0 ถึง 18.4 quintillion
    let k: u128 = u128::MAX;
    let l: usize = 42;        // ขนาดของ pointer

    println!("i8 max: {}", i8::MAX);
    println!("i8 min: {}", i8::MIN);
    println!("u8 max: {}", u8::MAX);
    println!("i32 max: {}", i32::MAX);
    println!("u32 max: {}", u32::MAX);

    // ----------------------------
    // Integer Literals (รูปแบบต่างๆ)
    // ----------------------------
    let decimal     = 1_000_000;     // 1000000 (underscore เพื่อความอ่านง่าย)
    let hex         = 0xFF;          // 255
    let octal       = 0o77;          // 63
    let binary      = 0b1111_0000;   // 240
    let byte        = b'A';          // 65 (u8 เท่านั้น)

    println!("decimal = {}", decimal);
    println!("hex = {} (0xFF)", hex);
    println!("octal = {} (0o77)", octal);
    println!("binary = {} (0b1111_0000)", binary);
    println!("byte = {} (b'A')", byte);

    // ----------------------------
    // Integer Operations
    // ----------------------------
    let sum = 5 + 10;
    let diff = 95 - 4;
    let product = 4 * 30;
    let quotient = 56 / 32;      // integer division = 1
    let remainder = 43 % 5;      // 3

    println!("{} + {} = {}", 5, 10, sum);
    println!("{} - {} = {}", 95, 4, diff);
    println!("{} * {} = {}", 4, 30, product);
    println!("{} / {} = {}", 56, 32, quotient);
    println!("{} % {} = {}", 43, 5, remainder);

    // ----------------------------
    // Overflow Handling
    // ----------------------------
    let max_u8: u8 = 255;

    // Debug mode: panic on overflow
    // Release mode: wrapping by default

    // Explicit overflow methods
    let wrapped = max_u8.wrapping_add(1);    // 0 (wraps around)
    let checked = max_u8.checked_add(1);     // None (overflow)
    let saturated = max_u8.saturating_add(1); // 255 (stay at max)
    let overflowed = max_u8.overflowing_add(1); // (0, true)

    println!("wrapping: {}", wrapped);
    println!("checked: {:?}", checked);
    println!("saturated: {}", saturated);
    println!("overflowed: {:?}", overflowed);
}
```

### 5.2 Floating Point Types

```rust
fn main() {
    // f32 (32-bit) และ f64 (64-bit)
    let x: f32 = 3.14;
    let y: f64 = 3.141592653589793;  // default

    println!("f32: {}", x);
    println!("f64: {}", y);

    // Float precision
    println!("f32 precision: {:.10}", x);  // 3.1400001049
    println!("f64 precision: {:.10}", y);  // 3.1415926536

    // Special values
    let infinity = f64::INFINITY;
    let neg_infinity = f64::NEG_INFINITY;
    let nan = f64::NAN;
    let zero = 0.0_f64;

    println!("infinity: {}", infinity);
    println!("-infinity: {}", neg_infinity);
    println!("NaN: {}", nan);
    println!("is NaN: {}", nan.is_nan());
    println!("is infinite: {}", infinity.is_infinite());
    println!("is finite: {}", (1.0_f64).is_finite());

    // Float operations
    let a = 2.0_f64;
    let b = 3.0_f64;

    println!("sqrt({}) = {}", a, a.sqrt());          // 1.414...
    println!("{}^{} = {}", a, b, a.powf(b));         // 8.0
    println!("log({}) = {:.4}", a, a.ln());           // 0.6931
    println!("log10({}) = {:.4}", 100.0_f64, 100.0_f64.log10()); // 2.0

    // Rounding
    let num = 3.7_f64;
    println!("floor({}) = {}", num, num.floor());  // 3.0
    println!("ceil({}) = {}", num, num.ceil());    // 4.0
    println!("round({}) = {}", num, num.round());  // 4.0
    println!("trunc({}) = {}", num, num.trunc());  // 3.0

    // ระวัง: Float comparison
    let x = 0.1_f64 + 0.2_f64;
    println!("0.1 + 0.2 = {}", x);           // 0.30000000000000004
    println!("0.1 + 0.2 == 0.3: {}", x == 0.3); // false!

    // ใช้ epsilon แทน
    let epsilon = f64::EPSILON;
    println!("0.1 + 0.2 ≈ 0.3: {}", (x - 0.3).abs() < epsilon * 10.0);

    // Float constants
    println!("π = {}", std::f64::consts::PI);
    println!("e = {}", std::f64::consts::E);
    println!("√2 = {}", std::f64::consts::SQRT_2);
}
```

### 5.3 Boolean Type

```rust
fn main() {
    let t: bool = true;
    let f: bool = false;

    println!("t = {}", t);
    println!("f = {}", f);
    println!("size of bool: {} bytes", std::mem::size_of::<bool>());

    // Boolean operations
    println!("true AND true = {}", t && t);
    println!("true AND false = {}", t && f);
    println!("true OR false = {}", t || f);
    println!("NOT true = {}", !t);

    // Short-circuit evaluation
    let x = 0;
    let result = x != 0 && 10 / x > 2;  // x != 0 is false, so 10/x never runs
    println!("Short-circuit: {}", result);

    // Boolean in conditions
    let is_rust_fast = true;
    let is_rust_safe = true;

    if is_rust_fast && is_rust_safe {
        println!("Rust เร็วและปลอดภัย!");
    }

    // Boolean comparison
    let age = 25;
    let is_adult = age >= 18;
    println!("is adult: {}", is_adult);

    // Boolean to int
    let score = true as i32;  // 1
    let no_score = false as i32;  // 0
    println!("true as i32: {}", score);
    println!("false as i32: {}", no_score);
}
```

### 5.4 Character Type

```rust
fn main() {
    // char เป็น Unicode scalar value (4 bytes)
    let c1: char = 'A';
    let c2: char = 'ก';     // ภาษาไทย
    let c3: char = '🦀';    // emoji!
    let c4: char = '中';    // Chinese
    let c5: char = '\n';    // newline
    let c6: char = '\t';    // tab
    let c7: char = '\\';    // backslash
    let c8: char = '\'';    // single quote

    println!("c1 = {}", c1);
    println!("c2 = {}", c2);
    println!("c3 = {}", c3);
    println!("c4 = {}", c4);
    println!("size of char: {} bytes", std::mem::size_of::<char>());

    // Character methods
    let ch = 'A';
    println!("is alphabetic: {}", ch.is_alphabetic());  // true
    println!("is uppercase: {}", ch.is_uppercase());    // true
    println!("is digit: {}", ch.is_ascii_digit());      // false
    println!("to lowercase: {}", ch.to_lowercase().next().unwrap()); // 'a'

    let digit = '5';
    println!("is digit: {}", digit.is_ascii_digit());  // true
    println!("to digit value: {}", digit.to_digit(10).unwrap()); // 5

    // char to/from u32
    let ascii_val = 'A' as u32;
    println!("'A' as u32 = {}", ascii_val);  // 65

    let from_u32 = char::from_u32(65).unwrap();
    println!("char from 65 = {}", from_u32);  // 'A'

    // Iterate through chars
    let text = "Hello, 🌍!";
    println!("Characters in '{}': ", text);
    for c in text.chars() {
        print!("[{}]", c);
    }
    println!();
}
```

---

## 6. Compound Types

### 6.1 Tuples

```rust
fn main() {
    // Tuple: ขนาดคงที่, ชนิดต่างกันได้
    let tup: (i32, f64, bool, char) = (500, 6.4, true, 'Z');

    // Destructuring
    let (x, y, z, w) = tup;
    println!("x={}, y={}, z={}, w={}", x, y, z, w);

    // Index access
    println!("tup.0 = {}", tup.0);
    println!("tup.1 = {}", tup.1);
    println!("tup.2 = {}", tup.2);
    println!("tup.3 = {}", tup.3);

    // Unit tuple () - "empty" return type
    let unit: () = ();

    // Tuple ใน function
    let point = (3, 4);
    let distance = calculate_distance(point);
    println!("Distance: {:.2}", distance);

    // Return multiple values
    let (min, max) = find_min_max(&[3, 1, 4, 1, 5, 9, 2, 6]);
    println!("min={}, max={}", min, max);

    // Nested tuples
    let nested = ((1, 2), (3, 4));
    println!("nested.0.0 = {}", nested.0.0);
    println!("nested.1.1 = {}", nested.1.1);

    // Named via struct (better for readability)
    // แต่ถ้าเป็น temporary เล็กๆ tuple ก็โอเค
    let rgb = (255u8, 128u8, 0u8);
    println!("R={}, G={}, B={}", rgb.0, rgb.1, rgb.2);
}

fn calculate_distance(point: (f64, f64)) -> f64 {
    let (x, y) = point;
    (x * x + y * y).sqrt()
}

fn find_min_max(numbers: &[i32]) -> (i32, i32) {
    let mut min = numbers[0];
    let mut max = numbers[0];

    for &n in numbers {
        if n < min { min = n; }
        if n > max { max = n; }
    }

    (min, max)
}
```

### 6.2 Arrays

```rust
fn main() {
    // Array: ขนาดคงที่ ณ compile time, ชนิดเดียวกัน
    let arr: [i32; 5] = [1, 2, 3, 4, 5];
    println!("arr = {:?}", arr);
    println!("arr[0] = {}", arr[0]);
    println!("arr length = {}", arr.len());

    // Initialize ด้วยค่าเดียวกัน
    let zeros = [0; 10];  // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
    println!("zeros = {:?}", zeros);

    let fives = [5; 3];   // [5, 5, 5]
    println!("fives = {:?}", fives);

    // Array slice
    let slice = &arr[1..4];  // [2, 3, 4]
    println!("slice = {:?}", slice);

    // Iterate
    for element in &arr {
        print!("{} ", element);
    }
    println!();

    // Iterate with index
    for (i, val) in arr.iter().enumerate() {
        println!("arr[{}] = {}", i, val);
    }

    // Mutable array
    let mut data = [1, 2, 3, 4, 5];
    data[2] = 99;
    println!("data = {:?}", data);  // [1, 2, 99, 4, 5]

    // 2D array
    let matrix: [[i32; 3]; 3] = [
        [1, 2, 3],
        [4, 5, 6],
        [7, 8, 9],
    ];

    println!("Matrix:");
    for row in &matrix {
        for &val in row {
            print!("{:3}", val);
        }
        println!();
    }

    // Array methods
    let numbers = [3, 1, 4, 1, 5, 9, 2, 6];
    println!("contains 5: {}", numbers.contains(&5));
    println!("first: {:?}", numbers.first());
    println!("last: {:?}", numbers.last());

    // Sort
    let mut sortable = [3, 1, 4, 1, 5, 9, 2, 6];
    sortable.sort();
    println!("sorted: {:?}", sortable);

    // Pattern matching with arrays
    let rgb = [255, 128, 0];
    match rgb {
        [r, g, b] => println!("R={}, G={}, B={}", r, g, b),
    }

    // Bounds checking (สำคัญมาก!)
    // let out_of_bounds = arr[10];  // PANIC at runtime!
    // ใช้ get() แทนเพื่อความปลอดภัย
    match arr.get(10) {
        Some(val) => println!("value: {}", val),
        None => println!("Index out of bounds!"),
    }
}
```

---

## 7. Type Conversion (Type Casting)

### 7.1 Numeric Conversions

```rust
fn main() {
    // as keyword สำหรับ primitive type conversion
    let x: i32 = 42;
    let y = x as f64;  // i32 → f64
    let z = y as i32;  // f64 → i32 (ทิ้งส่วนทศนิยม)

    println!("i32: {}", x);
    println!("as f64: {}", y);
    println!("back to i32: {}", z);

    // Integer size conversions
    let big: i64 = 1000;
    let small = big as i32;   // safe (value fits)
    let tiny = big as i8;     // 1000 % 256 = 232 (truncates!)

    println!("i64: {}", big);
    println!("as i32: {}", small);
    println!("as i8: {}", tiny);  // 232 (might be unexpected!)

    // Float to int
    let f = 3.99_f64;
    let i = f as i32;  // truncates, not rounds!
    println!("{} as i32 = {}", f, i);  // 3 (not 4!)

    // Overflow
    let big_float = 300.0_f32;
    let as_u8 = big_float as u8;  // 255 (saturates)
    println!("{} as u8 = {}", big_float, as_u8);

    // Negative to unsigned
    let neg: i32 = -1;
    let as_u32 = neg as u32;  // wraps!
    println!("{} as u32 = {}", neg, as_u32);  // 4294967295

    // char to/from integers
    let ch = 'A';
    let num = ch as u32;
    println!("'{}' as u32 = {}", ch, num);  // 65

    let num: u8 = 65;
    let ch = num as char;
    println!("{} as char = '{}'", num, ch);  // 'A'

    // Boolean
    let b = true;
    let num = b as i32;
    println!("true as i32 = {}", num);  // 1
}
```

### 7.2 Type Conversion แบบ Safe

```rust
fn main() {
    // From/Into traits (safe conversion)
    let s = String::from("hello");     // &str → String
    let n = i64::from(42i32);          // i32 → i64

    println!("s = {}", s);
    println!("n = {}", n);

    // Into (reverse of From)
    let x: i64 = 42i32.into();
    println!("x = {}", x);

    // TryFrom/TryInto (fallible conversion)
    use std::convert::TryFrom;

    let big: i32 = 1000;
    match i8::try_from(big) {
        Ok(small) => println!("Converted: {}", small),
        Err(e) => println!("Conversion failed: {}", e),
    }

    let ok: i32 = 100;
    match i8::try_from(ok) {
        Ok(small) => println!("Converted: {}", small),
        Err(e) => println!("Conversion failed: {}", e),
    }

    // Parse from string
    let num_str = "42";
    let num: i32 = num_str.parse().unwrap();
    println!("parsed: {}", num);

    // Parse with error handling
    let bad_str = "not a number";
    match bad_str.parse::<i32>() {
        Ok(n) => println!("Parsed: {}", n),
        Err(e) => println!("Parse error: {}", e),
    }

    // ToString
    let n = 42;
    let s = n.to_string();
    println!("to_string: '{}'", s);

    // format! macro
    let formatted = format!("{:05}", 42);
    println!("formatted: '{}'", formatted);  // "00042"
}
```

---

## 8. Type Inference

Rust มี type inference ที่ฉลาดมาก:

```rust
fn main() {
    // Rust อนุมาน type จาก value
    let x = 42;         // i32 (default integer)
    let y = 3.14;       // f64 (default float)
    let z = true;       // bool
    let s = "hello";    // &str
    let owned = String::from("hello"); // String

    // Inference จาก context
    let mut numbers = Vec::new();  // ยังไม่รู้ type
    numbers.push(1);               // ตอนนี้รู้แล้วว่า Vec<i32>
    numbers.push(2);
    println!("{:?}", numbers);

    // Inference จาก return type ที่ต้องการ
    let parsed: i32 = "42".parse().unwrap();
    println!("parsed = {}", parsed);

    // Type annotation เมื่อ compiler ไม่รู้
    let numbers: Vec<i32> = vec![1, 2, 3];
    let empty: Vec<String> = Vec::new();

    // Turbofish syntax
    let parsed = "42".parse::<i32>().unwrap();
    let collected = vec!["1", "2", "3"]
        .into_iter()
        .map(|s| s.parse::<i32>().unwrap())
        .collect::<Vec<_>>();
    println!("{:?}", collected);

    // Generic type inference
    fn largest<T: PartialOrd>(list: &[T]) -> &T {
        let mut largest = &list[0];
        for item in list {
            if item > largest {
                largest = item;
            }
        }
        largest
    }

    let numbers = vec![34, 50, 25, 100, 65];
    let result = largest(&numbers);  // Rust infers T = i32
    println!("Largest number: {}", result);

    let chars = vec!['y', 'm', 'a', 'q'];
    let result = largest(&chars);    // Rust infers T = char
    println!("Largest char: {}", result);
}
```

---

## 9. ขนาดของชนิดข้อมูล

```rust
use std::mem;

fn main() {
    println!("=== ขนาดชนิดข้อมูล (bytes) ===");
    println!("bool:  {} byte",  mem::size_of::<bool>());
    println!("i8:    {} byte",  mem::size_of::<i8>());
    println!("i16:   {} bytes", mem::size_of::<i16>());
    println!("i32:   {} bytes", mem::size_of::<i32>());
    println!("i64:   {} bytes", mem::size_of::<i64>());
    println!("i128:  {} bytes", mem::size_of::<i128>());
    println!("isize: {} bytes", mem::size_of::<isize>());
    println!("u8:    {} byte",  mem::size_of::<u8>());
    println!("f32:   {} bytes", mem::size_of::<f32>());
    println!("f64:   {} bytes", mem::size_of::<f64>());
    println!("char:  {} bytes", mem::size_of::<char>());  // 4 bytes (Unicode)
    println!("&str:  {} bytes", mem::size_of::<&str>());  // 16 bytes (fat pointer)
    println!("String:{} bytes", mem::size_of::<String>()); // 24 bytes
    println!();

    // เปรียบเทียบกับ alignment
    println!("=== Alignment (bytes) ===");
    println!("i32 align: {}", mem::align_of::<i32>());
    println!("f64 align: {}", mem::align_of::<f64>());
    println!("bool align: {}", mem::align_of::<bool>());
}
```

**ผลลัพธ์:**
```
=== ขนาดชนิดข้อมูล (bytes) ===
bool:  1 byte
i8:    1 byte
i16:   2 bytes
i32:   4 bytes
i64:   8 bytes
i128:  16 bytes
isize: 8 bytes (on 64-bit)
u8:    1 byte
f32:   4 bytes
f64:   8 bytes
char:  4 bytes
&str:  16 bytes
String: 24 bytes
```

---

## 10. ตัวอย่างโปรแกรมจริง

### 10.1 Temperature Converter

```rust
// src/main.rs
fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - 32.0) * 5.0 / 9.0
}

fn celsius_to_kelvin(c: f64) -> f64 {
    c + 273.15
}

fn main() {
    let temperatures_c: [f64; 5] = [0.0, 20.0, 37.0, 100.0, -40.0];

    println!("{:>10} {:>12} {:>10}", "Celsius", "Fahrenheit", "Kelvin");
    println!("{}", "─".repeat(34));

    for &c in &temperatures_c {
        let f = celsius_to_fahrenheit(c);
        let k = celsius_to_kelvin(c);
        println!("{:>10.1}°C {:>10.1}°F {:>8.2}K", c, f, k);
    }

    println!();
    println!("ทดสอบแปลงกลับ:");
    let f = 98.6;
    let c = fahrenheit_to_celsius(f);
    println!("{:.1}°F = {:.2}°C", f, c);
}
```

### 10.2 BMI Calculator

```rust
#[derive(Debug)]
struct Person {
    name: String,
    weight_kg: f64,
    height_m: f64,
}

impl Person {
    fn new(name: &str, weight_kg: f64, height_m: f64) -> Self {
        Self {
            name: name.to_string(),
            weight_kg,
            height_m,
        }
    }

    fn bmi(&self) -> f64 {
        self.weight_kg / (self.height_m * self.height_m)
    }

    fn bmi_category(&self) -> &str {
        match self.bmi() {
            bmi if bmi < 18.5 => "น้ำหนักน้อยกว่าเกณฑ์",
            bmi if bmi < 25.0 => "น้ำหนักปกติ",
            bmi if bmi < 30.0 => "น้ำหนักเกิน",
            _ => "อ้วน",
        }
    }
}

fn main() {
    let people = vec![
        Person::new("สมชาย", 70.0, 1.75),
        Person::new("สมหญิง", 55.0, 1.60),
        Person::new("สมศรี", 90.0, 1.68),
    ];

    println!("{:<10} {:>8} {:>8} {:>8} {}", 
             "ชื่อ", "น้ำหนัก", "ส่วนสูง", "BMI", "หมวด");
    println!("{}", "─".repeat(55));

    for person in &people {
        println!("{:<10} {:>7.1}kg {:>7.2}m {:>7.2} {}",
                 person.name,
                 person.weight_kg,
                 person.height_m,
                 person.bmi(),
                 person.bmi_category());
    }
}
```

---

## 11. สรุปและ Exercises

### 11.1 สิ่งที่เรียนรู้

✅ Immutable vs Mutable variables  
✅ Constants และ Static variables  
✅ Shadowing  
✅ Integer types (i8/u8 ถึง i128/u128)  
✅ Float types (f32, f64)  
✅ Boolean type  
✅ Character type (Unicode)  
✅ Tuples และ Arrays  
✅ Type conversion (as, From/Into, parse)  
✅ Type inference  

### 11.2 Exercises

**Exercise 1: Variable Practice**
```rust
// สร้างตัวแปรแต่ละชนิด และแสดงผล
// พร้อมขนาด memory ที่ใช้
```

**Exercise 2: Unit Converter**
```rust
// สร้าง functions สำหรับแปลงหน่วย:
// - กิโลกรัม ↔ ปอนด์
// - เมตร ↔ ฟุต
// - กิโลเมตร ↔ ไมล์
```

**Exercise 3: Statistics Calculator**
```rust
// รับ array ของตัวเลข แล้วคำนวณ:
// - ค่าเฉลี่ย (mean)
// - ค่ากลาง (median)
// - ค่าสูงสุดและต่ำสุด
// - ส่วนเบี่ยงเบนมาตรฐาน (standard deviation)
```

### 11.3 แนวทาง Exercise 3

```rust
fn mean(numbers: &[f64]) -> f64 {
    numbers.iter().sum::<f64>() / numbers.len() as f64
}

fn median(numbers: &mut Vec<f64>) -> f64 {
    numbers.sort_by(|a, b| a.partial_cmp(b).unwrap());
    let mid = numbers.len() / 2;
    if numbers.len() % 2 == 0 {
        (numbers[mid - 1] + numbers[mid]) / 2.0
    } else {
        numbers[mid]
    }
}

fn std_deviation(numbers: &[f64]) -> f64 {
    let m = mean(numbers);
    let variance = numbers.iter()
        .map(|x| (x - m).powi(2))
        .sum::<f64>() / numbers.len() as f64;
    variance.sqrt()
}

fn main() {
    let mut data = vec![4.0, 8.0, 6.0, 5.0, 3.0, 2.0, 8.0, 9.0, 2.0, 5.0];

    println!("Data: {:?}", data);
    println!("Mean: {:.2}", mean(&data));
    println!("Median: {:.2}", median(&mut data));
    println!("Max: {}", data.iter().cloned().fold(f64::NEG_INFINITY, f64::max));
    println!("Min: {}", data.iter().cloned().fold(f64::INFINITY, f64::min));
    println!("Std Dev: {:.2}", std_deviation(&data));
}
```

---

*[← Part 001: การติดตั้งและ Hello World](../part_001/README.md) | [Part 003: Functions และ Control Flow →](../part_003/README.md)*
