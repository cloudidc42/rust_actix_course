# Part 003: Functions, Control Flow, และ Loops 🔄

## 🎯 เป้าหมายของ Part นี้

- เขียน functions แบบต่างๆ ได้
- ใช้ if/else, match, loop, while, for
- เข้าใจ expressions vs statements
- ใช้ nested control flow
- เขียน recursive functions

---

## 1. Functions

### 1.1 พื้นฐาน Functions

```rust
// fn keyword + ชื่อ function + parameters + return type
fn greet(name: &str, age: u32) -> String {
    format!("สวัสดี {}! คุณอายุ {} ปี", name, age)
}

// Function ที่ไม่ return ค่า (return unit type ())
fn print_separator(char: char, length: usize) {
    let line: String = std::iter::repeat(char).take(length).collect();
    println!("{}", line);
}

// Function ที่ return ค่าสุดท้ายโดยไม่ใช้ return keyword
fn square(x: i32) -> i32 {
    x * x  // expression ไม่มี semicolon = return value
}

// Function ที่ใช้ explicit return
fn absolute(x: i32) -> i32 {
    if x < 0 {
        return -x;  // early return
    }
    x  // implicit return
}

fn main() {
    let msg = greet("สมชาย", 25);
    println!("{}", msg);

    print_separator('=', 40);
    println!("square(5) = {}", square(5));
    println!("absolute(-7) = {}", absolute(-7));
    println!("absolute(3) = {}", absolute(3));
}
```

### 1.2 Multiple Return Values

```rust
// Return tuple
fn divide(a: f64, b: f64) -> (f64, bool) {
    if b == 0.0 {
        (0.0, false)
    } else {
        (a / b, true)
    }
}

// Return Result type (แนะนำมากกว่า)
fn safe_divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("หารด้วย 0 ไม่ได้"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    let (result, ok) = divide(10.0, 3.0);
    if ok {
        println!("10 / 3 = {:.4}", result);
    }

    let (result, ok) = divide(10.0, 0.0);
    if !ok {
        println!("Division failed");
    }

    // ใช้ Result
    match safe_divide(10.0, 3.0) {
        Ok(r) => println!("Result: {:.4}", r),
        Err(e) => println!("Error: {}", e),
    }

    match safe_divide(10.0, 0.0) {
        Ok(r) => println!("Result: {:.4}", r),
        Err(e) => println!("Error: {}", e),
    }
}
```

### 1.3 Function Parameters แบบต่างๆ

```rust
// Borrow (reference)
fn calculate_sum(numbers: &[i32]) -> i32 {
    numbers.iter().sum()
}

// Mutable reference
fn double_all(numbers: &mut Vec<i32>) {
    for n in numbers.iter_mut() {
        *n *= 2;
    }
}

// Ownership transfer
fn consume_string(s: String) -> usize {
    s.len()  // s ถูกทำลายเมื่อออกจาก function
}

// Clone (copy)
fn process_name(name: String) -> String {
    format!("คุณ{}", name)
}

fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
    let sum = calculate_sum(&numbers);  // borrow
    println!("sum = {}", sum);
    println!("numbers still accessible: {:?}", numbers);  // ยังใช้ได้

    let mut nums = vec![1, 2, 3];
    double_all(&mut nums);  // mutable borrow
    println!("doubled: {:?}", nums);

    let s = String::from("Hello");
    let len = consume_string(s);  // s ถูก move แล้ว
    // println!("{}", s);  // ERROR! s ถูก move ไปแล้ว
    println!("length was: {}", len);

    let name = String::from("สมชาย");
    let name_clone = name.clone();
    let processed = process_name(name);  // name ถูก move
    println!("{}", processed);
    println!("original clone: {}", name_clone);
}
```

### 1.4 Default Parameter Values (ผ่าน Builder Pattern)

```rust
// Rust ไม่มี default parameters โดยตรง
// แต่ใช้ Builder pattern หรือ Option ได้

fn greet_with_options(name: &str, title: Option<&str>, times: Option<u32>) {
    let title = title.unwrap_or("");
    let times = times.unwrap_or(1);

    for _ in 0..times {
        if title.is_empty() {
            println!("สวัสดี {}!", name);
        } else {
            println!("สวัสดี{} {}!", title, name);
        }
    }
}

fn main() {
    greet_with_options("สมชาย", None, None);
    greet_with_options("สมหญิง", Some("คุณ"), None);
    greet_with_options("สมศรี", Some("ดร."), Some(3));
}
```

---

## 2. Control Flow

### 2.1 if / else if / else

```rust
fn grade(score: u32) -> &'static str {
    if score >= 90 {
        "A"
    } else if score >= 80 {
        "B"
    } else if score >= 70 {
        "C"
    } else if score >= 60 {
        "D"
    } else {
        "F"
    }
}

fn main() {
    // Basic if/else
    let x = 10;

    if x > 5 {
        println!("x มากกว่า 5");
    } else {
        println!("x น้อยกว่าหรือเท่ากับ 5");
    }

    // if เป็น expression ได้
    let status = if x > 0 { "positive" } else { "non-positive" };
    println!("x is {}", status);

    // ใช้ในการ assign
    let number = 7;
    let result = if number % 2 == 0 {
        format!("{} เป็นเลขคู่", number)
    } else {
        format!("{} เป็นเลขคี่", number)
    };
    println!("{}", result);

    // Grade calculator
    let scores = [95, 82, 71, 65, 45];
    for score in scores {
        println!("คะแนน {} = เกรด {}", score, grade(score));
    }

    // Nested if
    let age = 20;
    let has_id = true;

    if age >= 18 {
        if has_id {
            println!("เข้าได้ (มีบัตรประชาชน)");
        } else {
            println!("ต้องแสดงบัตรประชาชน");
        }
    } else {
        println!("อายุน้อยเกินไป");
    }

    // แบบ idiomatic Rust
    if age >= 18 && has_id {
        println!("เข้าได้");
    } else {
        println!("เข้าไม่ได้");
    }
}
```

### 2.2 match Expression

```rust
fn describe_number(n: i32) -> &'static str {
    match n {
        0 => "ศูนย์",
        1 => "หนึ่ง",
        2 | 3 | 5 | 7 | 11 => "จำนวนเฉพาะ",
        13..=19 => "วัยรุ่น",
        n if n < 0 => "ลบ",
        n if n % 2 == 0 => "เลขคู่",
        _ => "เลขคี่อื่นๆ",
    }
}

fn main() {
    // Basic match
    let coin = "บาท";
    let value = match coin {
        "สตางค์" => 0.01,
        "บาท" => 1.0,
        "สิบบาท" => 10.0,
        _ => 0.0,
    };
    println!("{} = {} บาท", coin, value);

    // Match with binding
    let numbers = vec![0, 1, 2, 3, 5, 7, 13, 15, 20, -5];
    for n in numbers {
        println!("{}: {}", n, describe_number(n));
    }

    // Match tuple
    let point = (1, -1);
    match point {
        (0, 0) => println!("Origin"),
        (x, 0) => println!("On x-axis at {}", x),
        (0, y) => println!("On y-axis at {}", y),
        (x, y) if x == y => println!("On diagonal at ({}, {})", x, y),
        (x, y) => println!("Point at ({}, {})", x, y),
    }

    // Match enum
    #[derive(Debug)]
    enum Direction {
        North,
        South,
        East,
        West,
    }

    let dir = Direction::North;
    let arrow = match dir {
        Direction::North => "↑",
        Direction::South => "↓",
        Direction::East => "→",
        Direction::West => "←",
    };
    println!("Direction: {}", arrow);

    // Match with guard
    let pair = (2, -2);
    match pair {
        (x, y) if x == y => println!("Equal: {}", x),
        (x, y) if x + y == 0 => println!("Sum is zero: {} + {} = 0", x, y),
        (x, _) if x % 2 == 0 => println!("First is even: {}", x),
        _ => println!("No match"),
    }

    // Match with @ binding
    let n = 15;
    match n {
        x @ 1..=10 => println!("{} อยู่ระหว่าง 1-10", x),
        x @ 11..=20 => println!("{} อยู่ระหว่าง 11-20", x),
        x => println!("{} อยู่นอกช่วง", x),
    }

    // Destructuring in match
    struct Point { x: i32, y: i32 }

    let p = Point { x: 3, y: 5 };
    match p {
        Point { x: 0, y } => println!("On y-axis at y={}", y),
        Point { x, y: 0 } => println!("On x-axis at x={}", x),
        Point { x, y } => println!("At ({}, {})", x, y),
    }
}
```

### 2.3 if let และ while let

```rust
fn main() {
    // if let - สำหรับ match แบบ single pattern
    let some_value: Option<i32> = Some(42);

    // แบบ verbose (match)
    match some_value {
        Some(x) => println!("Got: {}", x),
        None => println!("Nothing"),
    }

    // แบบ concise (if let)
    if let Some(x) = some_value {
        println!("Got: {}", x);
    }

    // if let with else
    let config_max: Option<u8> = Some(3u8);
    if let Some(max) = config_max {
        println!("Max is {}", max);
    } else {
        println!("No max configured");
    }

    // Nested if let
    let pair: Option<(i32, i32)> = Some((3, 7));
    if let Some((x, y)) = pair {
        println!("x={}, y={}", x, y);
    }

    // while let
    let mut stack = vec![1, 2, 3, 4, 5];
    print!("Popping: ");
    while let Some(top) = stack.pop() {
        print!("{} ", top);
    }
    println!();

    // while let กับ iterator
    let mut iter = vec!["a", "b", "c"].into_iter();
    while let Some(val) = iter.next() {
        print!("{} ", val);
    }
    println!();

    // if let chain (Rust 1.64+)
    let x: Option<i32> = Some(10);
    let y: Option<&str> = Some("hello");
    if let (Some(a), Some(b)) = (x, y) {
        println!("Both: {} {}", a, b);
    }
}
```

---

## 3. Loops

### 3.1 loop (infinite loop)

```rust
fn main() {
    // Basic loop
    let mut count = 0;
    loop {
        count += 1;
        if count == 5 {
            break;
        }
    }
    println!("count = {}", count);

    // loop ที่ return ค่า
    let result = loop {
        count += 1;
        if count == 10 {
            break count * 2;  // return ค่าผ่าน break
        }
    };
    println!("result = {}", result);  // 20

    // Nested loops กับ labels
    let mut found = (0, 0);
    'outer: for i in 0..5 {
        for j in 0..5 {
            if i + j == 6 {
                found = (i, j);
                break 'outer;  // break จาก outer loop
            }
        }
    }
    println!("Found: {:?}", found);

    // Continue กับ labels
    'outer: for i in 0..3 {
        for j in 0..3 {
            if j == 1 {
                continue 'outer;  // skip ไปที่ iteration ถัดไปของ outer
            }
            println!("({}, {})", i, j);
        }
    }
}
```

### 3.2 while Loop

```rust
fn is_prime(n: u64) -> bool {
    if n < 2 { return false; }
    if n == 2 { return true; }
    if n % 2 == 0 { return false; }

    let mut i = 3;
    while i * i <= n {
        if n % i == 0 {
            return false;
        }
        i += 2;
    }
    true
}

fn main() {
    // Basic while
    let mut n = 1;
    while n < 100 {
        n *= 2;
    }
    println!("First power of 2 >= 100: {}", n);

    // While กับ condition
    let mut fibonacci = vec![0u64, 1];
    while *fibonacci.last().unwrap() < 1000 {
        let len = fibonacci.len();
        let next = fibonacci[len-1] + fibonacci[len-2];
        fibonacci.push(next);
    }
    println!("Fibonacci < 1000: {:?}", fibonacci);

    // Prime numbers
    print!("Primes < 50: ");
    let mut num = 2u64;
    while num < 50 {
        if is_prime(num) {
            print!("{} ", num);
        }
        num += 1;
    }
    println!();

    // Collatz conjecture
    let mut n = 27u64;
    let mut steps = 0;
    print!("Collatz(27): ");
    while n != 1 {
        if steps < 10 {
            print!("{} → ", n);
        }
        n = if n % 2 == 0 { n / 2 } else { 3 * n + 1 };
        steps += 1;
    }
    println!("... 1 ({} steps)", steps);
}
```

### 3.3 for Loop

```rust
fn main() {
    // for กับ range
    for i in 0..5 {
        print!("{} ", i);  // 0 1 2 3 4
    }
    println!();

    for i in 0..=5 {
        print!("{} ", i);  // 0 1 2 3 4 5
    }
    println!();

    // for กับ step
    // Rust ไม่มี step ใน range โดยตรง ใช้ step_by
    for i in (0..20).step_by(3) {
        print!("{} ", i);  // 0 3 6 9 12 15 18
    }
    println!();

    // Reverse
    for i in (0..5).rev() {
        print!("{} ", i);  // 4 3 2 1 0
    }
    println!();

    // for กับ array
    let fruits = ["แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง"];
    for fruit in &fruits {
        println!("ผลไม้: {}", fruit);
    }

    // for กับ index (enumerate)
    for (i, fruit) in fruits.iter().enumerate() {
        println!("{}. {}", i + 1, fruit);
    }

    // for กับ Vec
    let numbers: Vec<i32> = (1..=10).collect();
    let sum: i32 = numbers.iter().sum();
    println!("Sum 1-10 = {}", sum);

    // for กับ HashMap
    use std::collections::HashMap;
    let mut scores: HashMap<&str, i32> = HashMap::new();
    scores.insert("Alice", 95);
    scores.insert("Bob", 87);
    scores.insert("Charlie", 92);

    for (name, score) in &scores {
        println!("{}: {}", name, score);
    }

    // for กับ String
    let text = "Hello, สวัสดี!";
    for ch in text.chars() {
        if ch.is_alphabetic() {
            print!("{}", ch);
        }
    }
    println!();

    // for กับ filter/map
    let evens: Vec<i32> = (1..=20)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .collect();
    println!("Squares of even numbers 1-20: {:?}", evens);

    // Nested for
    println!("\nมาตรา:");
    for i in 2..=5 {
        for j in 1..=10 {
            print!("{:4}", i * j);
        }
        println!();
    }
}
```

---

## 4. Recursive Functions

### 4.1 Recursion พื้นฐาน

```rust
fn factorial(n: u64) -> u64 {
    match n {
        0 | 1 => 1,
        _ => n * factorial(n - 1),
    }
}

fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

// Tail recursive version (ดีกว่าเพราะ stack ไม่ล้น)
fn factorial_tail(n: u64, accumulator: u64) -> u64 {
    match n {
        0 | 1 => accumulator,
        _ => factorial_tail(n - 1, n * accumulator),
    }
}

fn fibonacci_memo(n: u32, memo: &mut std::collections::HashMap<u32, u64>) -> u64 {
    if let Some(&cached) = memo.get(&n) {
        return cached;
    }
    let result = match n {
        0 => 0,
        1 => 1,
        _ => fibonacci_memo(n - 1, memo) + fibonacci_memo(n - 2, memo),
    };
    memo.insert(n, result);
    result
}

fn main() {
    // Factorial
    for i in 0..=10 {
        println!("{}! = {}", i, factorial(i));
    }

    // Fibonacci (slow without memoization)
    print!("Fibonacci: ");
    for i in 0..15 {
        print!("{} ", fibonacci(i));
    }
    println!();

    // Fibonacci with memoization (fast)
    let mut memo = std::collections::HashMap::new();
    print!("Fibonacci (memo): ");
    for i in 0..20 {
        print!("{} ", fibonacci_memo(i, &mut memo));
    }
    println!();

    // Factorial tail recursive
    println!("20! = {}", factorial_tail(20, 1));
}
```

### 4.2 Tree Traversal (Recursive)

```rust
#[derive(Debug)]
enum Tree {
    Leaf(i32),
    Node {
        value: i32,
        left: Box<Tree>,
        right: Box<Tree>,
    },
}

impl Tree {
    fn sum(&self) -> i32 {
        match self {
            Tree::Leaf(v) => *v,
            Tree::Node { value, left, right } => {
                value + left.sum() + right.sum()
            }
        }
    }

    fn depth(&self) -> usize {
        match self {
            Tree::Leaf(_) => 1,
            Tree::Node { left, right, .. } => {
                1 + left.depth().max(right.depth())
            }
        }
    }

    fn in_order(&self) -> Vec<i32> {
        match self {
            Tree::Leaf(v) => vec![*v],
            Tree::Node { value, left, right } => {
                let mut result = left.in_order();
                result.push(*value);
                result.extend(right.in_order());
                result
            }
        }
    }
}

fn main() {
    //         10
    //        /  \
    //       5    15
    //      / \
    //     3   7
    let tree = Tree::Node {
        value: 10,
        left: Box::new(Tree::Node {
            value: 5,
            left: Box::new(Tree::Leaf(3)),
            right: Box::new(Tree::Leaf(7)),
        }),
        right: Box::new(Tree::Leaf(15)),
    };

    println!("Sum: {}", tree.sum());          // 40
    println!("Depth: {}", tree.depth());      // 3
    println!("In-order: {:?}", tree.in_order()); // [3, 5, 7, 10, 15]
}
```

---

## 5. Advanced Pattern Matching

### 5.1 Destructuring

```rust
fn main() {
    // Destructure struct
    struct Point { x: f64, y: f64 }
    struct Rectangle { top_left: Point, bottom_right: Point }

    let rect = Rectangle {
        top_left: Point { x: 0.0, y: 10.0 },
        bottom_right: Point { x: 10.0, y: 0.0 },
    };

    let Rectangle {
        top_left: Point { x: x1, y: y1 },
        bottom_right: Point { x: x2, y: y2 },
    } = rect;

    let width = (x2 - x1).abs();
    let height = (y1 - y2).abs();
    println!("Width: {}, Height: {}, Area: {}", width, height, width * height);

    // Destructure tuple in for
    let pairs = [(1, 'a'), (2, 'b'), (3, 'c')];
    for (num, ch) in &pairs {
        println!("{}: {}", num, ch);
    }

    // Destructure enum
    #[derive(Debug)]
    enum Message {
        Quit,
        Move { x: i32, y: i32 },
        Write(String),
        ChangeColor(u8, u8, u8),
    }

    let messages = vec![
        Message::Move { x: 10, y: 20 },
        Message::Write(String::from("hello")),
        Message::ChangeColor(255, 0, 128),
        Message::Quit,
    ];

    for msg in &messages {
        match msg {
            Message::Quit => println!("Quit"),
            Message::Move { x, y } => println!("Move to ({}, {})", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
        }
    }
}
```

### 5.2 Complex Pattern Matching

```rust
fn process_command(input: &str) -> String {
    let parts: Vec<&str> = input.trim().splitn(2, ' ').collect();

    match parts.as_slice() {
        ["quit"] | ["exit"] => String::from("Goodbye!"),
        ["hello"] => String::from("สวัสดี!"),
        ["hello", name] => format!("สวัสดี, {}!", name),
        ["add", numbers] => {
            let sum: i32 = numbers
                .split(',')
                .filter_map(|s| s.trim().parse::<i32>().ok())
                .sum();
            format!("Sum = {}", sum)
        },
        ["help"] => String::from("Commands: quit, hello [name], add n1,n2,..."),
        [cmd, ..] => format!("Unknown command: {}", cmd),
        [] => String::from("Empty input"),
    }
}

fn main() {
    let commands = [
        "hello",
        "hello สมชาย",
        "add 1, 2, 3, 4, 5",
        "quit",
        "help",
        "unknown command",
    ];

    for cmd in &commands {
        println!("> {}", cmd);
        println!("  {}", process_command(cmd));
    }
}
```

---

## 6. Iterators และ Functional Style

### 6.1 Iterator Methods

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // map: transform each element
    let doubled: Vec<i32> = numbers.iter().map(|&x| x * 2).collect();
    println!("doubled: {:?}", doubled);

    // filter: keep matching elements
    let evens: Vec<&i32> = numbers.iter().filter(|&&x| x % 2 == 0).collect();
    println!("evens: {:?}", evens);

    // filter_map: filter + transform
    let strings = vec!["1", "two", "3", "four", "5"];
    let parsed: Vec<i32> = strings
        .iter()
        .filter_map(|s| s.parse().ok())
        .collect();
    println!("parsed numbers: {:?}", parsed);

    // fold: accumulate
    let sum = numbers.iter().fold(0, |acc, &x| acc + x);
    let product = numbers.iter().fold(1, |acc, &x| acc * x);
    println!("sum = {}, product = {}", sum, product);

    // reduce
    let max = numbers.iter().copied().reduce(|a, b| a.max(b));
    println!("max = {:?}", max);

    // any and all
    let has_even = numbers.iter().any(|&x| x % 2 == 0);
    let all_positive = numbers.iter().all(|&x| x > 0);
    println!("has even: {}, all positive: {}", has_even, all_positive);

    // find
    let first_gt5 = numbers.iter().find(|&&x| x > 5);
    println!("first > 5: {:?}", first_gt5);

    // position
    let pos = numbers.iter().position(|&x| x == 7);
    println!("position of 7: {:?}", pos);

    // count
    let even_count = numbers.iter().filter(|&&x| x % 2 == 0).count();
    println!("even count: {}", even_count);

    // take and skip
    let first3: Vec<&i32> = numbers.iter().take(3).collect();
    let skip3: Vec<&i32> = numbers.iter().skip(3).collect();
    println!("first 3: {:?}", first3);
    println!("skip 3: {:?}", skip3);

    // zip
    let a = vec![1, 2, 3];
    let b = vec!["one", "two", "three"];
    let zipped: Vec<_> = a.iter().zip(b.iter()).collect();
    println!("zipped: {:?}", zipped);

    // flatten
    let nested = vec![vec![1, 2], vec![3, 4], vec![5, 6]];
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("flattened: {:?}", flat);

    // chain
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let chained: Vec<&i32> = a.iter().chain(b.iter()).collect();
    println!("chained: {:?}", chained);

    // min, max
    println!("min: {:?}", numbers.iter().min());
    println!("max: {:?}", numbers.iter().max());

    // sum, product
    let sum: i32 = numbers.iter().sum();
    let product: i64 = (1i64..=10).product();
    println!("sum: {}, product: {}", sum, product);

    // Complex pipeline
    let result: String = (1..=20)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x)
        .take(5)
        .map(|x| x.to_string())
        .collect::<Vec<_>>()
        .join(", ");
    println!("squares of first 5 even numbers: {}", result);
}
```

---

## 7. โปรแกรมตัวอย่าง: Number Guessing Game

```rust
use std::io;
use std::io::Write;

fn get_random_number(min: u32, max: u32) -> u32 {
    // Simple LCG pseudo-random (ไม่ดีสำหรับ production แต่ใช้สำหรับตัวอย่าง)
    use std::time::{SystemTime, UNIX_EPOCH};
    let seed = SystemTime::now()
        .duration_since(UNIX_EPOCH)
        .unwrap()
        .subsec_nanos();
    (seed % (max - min + 1)) + min
}

fn play_game() {
    let secret = get_random_number(1, 100);
    let mut attempts = 0;
    let max_attempts = 7;

    println!("🎮 เกมทายตัวเลข");
    println!("ฉันคิดตัวเลขระหว่าง 1-100 ไว้แล้ว");
    println!("คุณมี {} ครั้งในการทาย", max_attempts);
    println!();

    loop {
        if attempts >= max_attempts {
            println!("❌ หมดโอกาสแล้ว! ตัวเลขคือ {}", secret);
            break;
        }

        print!("ครั้งที่ {} > ", attempts + 1);
        io::stdout().flush().unwrap();

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();

        let guess: u32 = match input.trim().parse() {
            Ok(n) if n >= 1 && n <= 100 => n,
            Ok(_) => {
                println!("กรุณาใส่ตัวเลขระหว่าง 1-100");
                continue;
            },
            Err(_) => {
                println!("กรุณาใส่ตัวเลข");
                continue;
            }
        };

        attempts += 1;
        let remaining = max_attempts - attempts;

        match guess.cmp(&secret) {
            std::cmp::Ordering::Less => {
                if remaining > 0 {
                    println!("📈 น้อยเกินไป! เหลือ {} ครั้ง", remaining);
                }
            },
            std::cmp::Ordering::Greater => {
                if remaining > 0 {
                    println!("📉 มากเกินไป! เหลือ {} ครั้ง", remaining);
                }
            },
            std::cmp::Ordering::Equal => {
                println!("🎉 ถูกต้อง! คุณทาย {} ครั้ง", attempts);
                break;
            },
        }
    }
}

fn main() {
    loop {
        play_game();
        println!();
        print!("เล่นอีกไหม? (y/n): ");
        io::stdout().flush().unwrap();

        let mut input = String::new();
        io::stdin().read_line(&mut input).unwrap();

        match input.trim().to_lowercase().as_str() {
            "y" | "yes" | "ใช่" => {
                println!();
                continue;
            },
            _ => {
                println!("ขอบคุณที่เล่น! บาย!");
                break;
            }
        }
    }
}
```

---

## 8. สรุปและ Exercises

### 8.1 สิ่งที่เรียนรู้

✅ Functions และ return values  
✅ if/else expressions  
✅ match patterns แบบต่างๆ  
✅ if let และ while let  
✅ loop, while, for loops  
✅ Recursive functions  
✅ Iterator methods (map, filter, fold)  
✅ Complex pattern matching  

### 8.2 Exercises

**Exercise 1: FizzBuzz**
```rust
// พิมพ์ 1-100:
// - หาร 3 ลงตัว → "Fizz"
// - หาร 5 ลงตัว → "Buzz"  
// - หาร 15 ลงตัว → "FizzBuzz"
// - อื่นๆ → ตัวเลข
```

**Exercise 2: Caesar Cipher**
```rust
// เข้ารหัส/ถอดรหัส Caesar cipher
// fn encode(text: &str, shift: i32) -> String
// fn decode(text: &str, shift: i32) -> String
```

**Exercise 3: Number Pyramid**
```rust
// พิมพ์รูปสามเหลี่ยมตัวเลข:
//     1
//    121
//   12321
//  1234321
// 123454321
```

---

*[← Part 002: ตัวแปรและชนิดข้อมูล](../part_002/README.md) | [Part 004: Ownership และ Borrowing →](../part_004/README.md)*
