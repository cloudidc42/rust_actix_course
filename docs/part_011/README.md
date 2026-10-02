# Part 011: Closures และ Iterators 🔄

## 🎯 เป้าหมายของ Part นี้

- เข้าใจ Closure syntax และการ capture variables
- Fn, FnMut, FnOnce traits
- Iterator trait และ custom iterators
- Iterator adapters และ consumers
- Lazy evaluation และการ chain operations
- สร้าง data processing pipeline

---

## 1. Closure พื้นฐาน

Closure คือ anonymous function ที่สามารถ capture ค่าจาก scope รอบข้างได้

```rust
fn main() {
    // Closure syntax พื้นฐาน
    let add_one = |x| x + 1;
    println!("{}", add_one(5));  // 6

    // Closure พร้อม type annotation
    let multiply: fn(i32, i32) -> i32 = |x, y| x * y;
    println!("{}", multiply(3, 4));  // 12

    // Multi-line closure
    let complex = |x: i32| {
        let y = x * 2;
        let z = y + 10;
        z  // implicit return
    };
    println!("{}", complex(5));  // 20

    // Closure ไม่มี parameter
    let greet = || println!("สวัสดี Rust!");
    greet();

    // Closure เป็น argument ให้ function อื่น
    let numbers = vec![1, 2, 3, 4, 5];
    let doubled: Vec<i32> = numbers.iter().map(|&x| x * 2).collect();
    println!("{:?}", doubled);  // [2, 4, 6, 8, 10]
}
```

---

## 2. Capturing Variables

```rust
fn main() {
    let x = 5;
    let y = 10;

    // Capture by immutable reference (default)
    let add_x = |n| n + x;
    println!("{}", add_x(3));   // 8
    println!("x ยังใช้ได้: {}", x);  // OK

    // Capture by mutable reference
    let mut count = 0;
    let mut increment = || {
        count += 1;
        println!("count = {}", count);
    };
    increment();  // count = 1
    increment();  // count = 2
    // println!("{}", count);  // ERROR: borrowed as mutable

    // Closure capture หลายตัวแปร
    let base = 100;
    let multiplier = 2;
    let calc = |n| n * multiplier + base;
    println!("{}", calc(5));  // 110

    // Capture struct
    #[derive(Debug)]
    struct Point { x: f64, y: f64 }
    
    let origin = Point { x: 0.0, y: 0.0 };
    let distance = |p: &Point| {
        ((p.x - origin.x).powi(2) + (p.y - origin.y).powi(2)).sqrt()
    };
    
    let p = Point { x: 3.0, y: 4.0 };
    println!("ระยะทาง: {}", distance(&p));  // 5.0
}
```

---

## 3. move Keyword

```rust
use std::thread;

fn main() {
    // move บังคับให้ closure เป็นเจ้าของ captured values
    let name = String::from("Rust");
    
    let greet = move || {
        println!("Hello, {}!", name);
    };
    
    greet();
    // println!("{}", name);  // ERROR: name ถูก move แล้ว
    
    // move สำคัญมากใน thread
    let data = vec![1, 2, 3, 4, 5];
    
    let handle = thread::spawn(move || {
        // data ถูก move เข้า thread นี้
        let sum: i32 = data.iter().sum();
        println!("Sum in thread: {}", sum);
    });
    
    handle.join().unwrap();
    // println!("{:?}", data);  // ERROR: data ถูก move แล้ว

    // move กับ Copy types - copy ไม่ใช่ move
    let n = 42;
    let show = move || println!("n = {}", n);
    show();
    println!("n ยังอยู่: {}", n);  // OK! i32 เป็น Copy

    // Returning closure จาก function
    fn make_adder(x: i32) -> impl Fn(i32) -> i32 {
        move |y| x + y  // x ถูก move เข้า closure
    }
    
    let add5 = make_adder(5);
    let add10 = make_adder(10);
    println!("{}", add5(3));   // 8
    println!("{}", add10(3));  // 13
}
```

---

## 4. Fn, FnMut, FnOnce Traits

```rust
// Fn: closure เรียกซ้ำได้ ไม่เปลี่ยน captured values
fn call_fn<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

// FnMut: closure เรียกซ้ำได้ แต่อาจเปลี่ยน captured values
fn call_fn_mut<F: FnMut(i32) -> i32>(mut f: F, x: i32) -> i32 {
    f(x)
}

// FnOnce: closure เรียกได้ครั้งเดียว (consume captured values)
fn call_once<F: FnOnce(i32) -> i32>(f: F, x: i32) -> i32 {
    f(x)
}

fn main() {
    // Fn - เรียกหลายครั้งได้
    let multiplier = 3;
    let triple = |x| x * multiplier;
    println!("{}", call_fn(triple, 5));   // 15
    println!("{}", call_fn(triple, 10));  // 30 - เรียกซ้ำได้

    // FnMut - modify captured variable
    let mut total = 0;
    let mut accumulate = |x: i32| {
        total += x;
        total
    };
    println!("{}", call_fn_mut(&mut accumulate, 5));   // 5
    println!("{}", call_fn_mut(&mut accumulate, 10));  // 15

    // FnOnce - consume captured value
    let name = String::from("World");
    let consume = move |prefix: i32| {
        format!("{}: Hello, {}!", prefix, name)  // name ถูก consume
    };
    println!("{}", call_once(consume, 1));
    // call_once(consume, 2);  // ERROR: consume ถูก consume แล้ว

    // Function pointers ก็ implement Fn traits
    fn double(x: i32) -> i32 { x * 2 }
    let result = call_fn(double, 7);
    println!("{}", result);  // 14

    // Storing closures in struct
    struct Counter {
        count: i32,
        on_increment: Box<dyn FnMut(i32)>,
    }
    
    impl Counter {
        fn new(callback: impl FnMut(i32) + 'static) -> Self {
            Counter {
                count: 0,
                on_increment: Box::new(callback),
            }
        }
        
        fn increment(&mut self) {
            self.count += 1;
            (self.on_increment)(self.count);
        }
    }
    
    let mut c = Counter::new(|n| println!("Count: {}", n));
    c.increment();  // Count: 1
    c.increment();  // Count: 2
    c.increment();  // Count: 3
}
```

---

## 5. Iterator Trait

```rust
// Iterator trait definition
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    // ... plus many default methods
}

// Custom iterator: Fibonacci
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { a: 0, b: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;
    
    fn next(&mut self) -> Option<u64> {
        let result = self.a;
        let next = self.a + self.b;
        self.a = self.b;
        self.b = next;
        Some(result)  // Fibonacci ไม่มีวันหมด
    }
}

// Custom iterator: Range with step
struct StepRange {
    current: i32,
    end: i32,
    step: i32,
}

impl StepRange {
    fn new(start: i32, end: i32, step: i32) -> Self {
        StepRange { current: start, end, step }
    }
}

impl Iterator for StepRange {
    type Item = i32;
    
    fn next(&mut self) -> Option<i32> {
        if self.current < self.end {
            let val = self.current;
            self.current += self.step;
            Some(val)
        } else {
            None
        }
    }
}

fn main() {
    // ใช้ Fibonacci iterator
    let fibs: Vec<u64> = Fibonacci::new().take(10).collect();
    println!("Fibonacci: {:?}", fibs);
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

    // หา Fibonacci ที่น้อยกว่า 100
    let small_fibs: Vec<u64> = Fibonacci::new()
        .take_while(|&n| n < 100)
        .collect();
    println!("Fibs < 100: {:?}", small_fibs);

    // ใช้ StepRange
    let evens: Vec<i32> = StepRange::new(0, 20, 2).collect();
    println!("Evens: {:?}", evens);  // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

    // Vec iterator ประเภทต่างๆ
    let v = vec![1, 2, 3, 4, 5];
    
    // iter() - yields &T (immutable references)
    for &x in v.iter() {
        print!("{} ", x);
    }
    println!();
    
    // iter_mut() - yields &mut T
    let mut v2 = vec![1, 2, 3];
    for x in v2.iter_mut() {
        *x *= 2;
    }
    println!("{:?}", v2);  // [2, 4, 6]
    
    // into_iter() - yields T (consumes collection)
    let v3 = vec![1, 2, 3];
    for x in v3.into_iter() {
        print!("{} ", x);
    }
    println!();
    // println!("{:?}", v3);  // ERROR: v3 consumed
}
```

---

## 6. Iterator Adapters

Adapters แปลง iterator หนึ่งเป็นอีก iterator (lazy - ไม่ทำงานจนกว่าจะ consume)

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // map - แปลงแต่ละ element
    let squares: Vec<i32> = numbers.iter()
        .map(|&x| x * x)
        .collect();
    println!("Squares: {:?}", squares);
    // [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

    // filter - กรอง elements
    let evens: Vec<&i32> = numbers.iter()
        .filter(|&&x| x % 2 == 0)
        .collect();
    println!("Evens: {:?}", evens);  // [2, 4, 6, 8, 10]

    // map + filter ร่วมกัน
    let even_squares: Vec<i32> = numbers.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x * x)
        .collect();
    println!("Even squares: {:?}", even_squares);  // [4, 16, 36, 64, 100]

    // flat_map - map แล้ว flatten
    let words = vec!["hello world", "foo bar baz"];
    let chars: Vec<&str> = words.iter()
        .flat_map(|s| s.split_whitespace())
        .collect();
    println!("Words: {:?}", chars);  // ["hello", "world", "foo", "bar", "baz"]

    let nested = vec![vec![1, 2, 3], vec![4, 5], vec![6, 7, 8, 9]];
    let flat: Vec<i32> = nested.into_iter()
        .flatten()
        .collect();
    println!("Flat: {:?}", flat);  // [1, 2, 3, 4, 5, 6, 7, 8, 9]

    // zip - รวม 2 iterators เป็น pairs
    let names = vec!["Alice", "Bob", "Charlie"];
    let scores = vec![95, 87, 92];
    let paired: Vec<(&&str, &i32)> = names.iter().zip(scores.iter()).collect();
    println!("Paired: {:?}", paired);

    // enumerate - เพิ่ม index
    for (i, name) in names.iter().enumerate() {
        println!("{}: {}", i, name);
    }

    // take - เอาแค่ N elements แรก
    let first_three: Vec<i32> = (1..=100).take(3).collect();
    println!("First 3: {:?}", first_three);  // [1, 2, 3]

    // skip - ข้าม N elements แรก
    let after_five: Vec<i32> = (1..=10).skip(5).collect();
    println!("After 5: {:?}", after_five);  // [6, 7, 8, 9, 10]

    // take_while / skip_while
    let until_five: Vec<i32> = (1..=10).take_while(|&x| x < 5).collect();
    println!("Until 5: {:?}", until_five);  // [1, 2, 3, 4]

    let from_five: Vec<i32> = (1..=10).skip_while(|&x| x < 5).collect();
    println!("From 5: {:?}", from_five);  // [5, 6, 7, 8, 9, 10]

    // chain - ต่อ 2 iterators
    let a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    let combined: Vec<i32> = a.iter().chain(b.iter()).copied().collect();
    println!("Combined: {:?}", combined);  // [1, 2, 3, 4, 5, 6]

    // peekable - ดู element ถัดไปโดยไม่ consume
    let mut iter = numbers.iter().peekable();
    while let Some(&next) = iter.peek() {
        if next > 5 { break; }
        iter.next();
        print!("{} ", next);
    }
    println!();  // 1 2 3 4 5

    // step_by
    let every_third: Vec<i32> = (0..20).step_by(3).collect();
    println!("Every 3rd: {:?}", every_third);  // [0, 3, 6, 9, 12, 15, 18]

    // rev - กลับลำดับ
    let reversed: Vec<i32> = (1..=5).rev().collect();
    println!("Reversed: {:?}", reversed);  // [5, 4, 3, 2, 1]

    // cloned / copied
    let refs: Vec<&i32> = numbers.iter().collect();
    let owned: Vec<i32> = refs.iter().copied().collect();
    println!("Owned: {:?}", &owned[..5]);
}
```

---

## 7. Iterator Consumers

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

    // collect - รวมเป็น collection
    let evens: Vec<i32> = numbers.iter()
        .filter(|&&x| x % 2 == 0)
        .copied()
        .collect();
    println!("Evens: {:?}", evens);

    // collect เป็น HashSet
    use std::collections::HashSet;
    let unique: HashSet<i32> = vec![1, 2, 2, 3, 3, 3].into_iter().collect();
    println!("Unique count: {}", unique.len());  // 3

    // collect เป็น HashMap
    use std::collections::HashMap;
    let map: HashMap<&str, i32> = vec![("a", 1), ("b", 2), ("c", 3)]
        .into_iter()
        .collect();
    println!("{:?}", map);

    // fold - accumulate ด้วย initial value
    let sum = numbers.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {}", sum);  // 55

    let product = numbers.iter().fold(1, |acc, &x| acc * x);
    println!("Product: {}", product);  // 3628800

    // สร้าง string จาก fold
    let words = vec!["hello", "world", "rust"];
    let sentence = words.iter().fold(String::new(), |mut acc, &word| {
        if !acc.is_empty() { acc.push(' '); }
        acc.push_str(word);
        acc
    });
    println!("Sentence: {}", sentence);  // hello world rust

    // sum / product
    let total: i32 = numbers.iter().sum();
    println!("Total: {}", total);  // 55

    let factorial: u64 = (1u64..=10).product();
    println!("10! = {}", factorial);  // 3628800

    // count
    let even_count = numbers.iter().filter(|&&x| x % 2 == 0).count();
    println!("Even count: {}", even_count);  // 5

    // any / all
    let has_even = numbers.iter().any(|&x| x % 2 == 0);
    println!("Has even: {}", has_even);  // true

    let all_positive = numbers.iter().all(|&x| x > 0);
    println!("All positive: {}", all_positive);  // true

    let all_less_than_5 = numbers.iter().all(|&x| x < 5);
    println!("All < 5: {}", all_less_than_5);  // false

    // find - หา element แรกที่ตรงเงื่อนไข
    let first_even = numbers.iter().find(|&&x| x % 2 == 0);
    println!("First even: {:?}", first_even);  // Some(2)

    let first_big = numbers.iter().find(|&&x| x > 100);
    println!("First > 100: {:?}", first_big);  // None

    // find_map - find และแปลงพร้อมกัน
    let first_even_square: Option<i32> = numbers.iter()
        .find_map(|&x| if x % 2 == 0 { Some(x * x) } else { None });
    println!("First even square: {:?}", first_even_square);  // Some(4)

    // position - หา index ของ element แรกที่ตรงเงื่อนไข
    let pos = numbers.iter().position(|&x| x == 7);
    println!("Position of 7: {:?}", pos);  // Some(6)

    // max / min
    println!("Max: {:?}", numbers.iter().max());  // Some(10)
    println!("Min: {:?}", numbers.iter().min());  // Some(1)

    // max_by_key / min_by_key
    let words2 = vec!["cat", "elephant", "dog", "rhinoceros"];
    let longest = words2.iter().max_by_key(|s| s.len());
    println!("Longest: {:?}", longest);  // Some("rhinoceros")

    // for_each - เหมือน for loop แต่ใช้ใน chain ได้
    numbers.iter()
        .filter(|&&x| x % 3 == 0)
        .for_each(|&x| print!("{} ", x));
    println!();  // 3 6 9

    // unzip - แยก Vec<(A, B)> เป็น (Vec<A>, Vec<B>)
    let pairs = vec![(1, 'a'), (2, 'b'), (3, 'c')];
    let (nums, chars): (Vec<i32>, Vec<char>) = pairs.into_iter().unzip();
    println!("Nums: {:?}, Chars: {:?}", nums, chars);

    // partition - แบ่ง collection ตาม predicate
    let (evens, odds): (Vec<i32>, Vec<i32>) = numbers.iter()
        .partition(|&&x| x % 2 == 0);
    println!("Evens: {:?}", evens);  // [2, 4, 6, 8, 10]
    println!("Odds: {:?}", odds);    // [1, 3, 5, 7, 9]
}
```

---

## 8. Lazy Evaluation

```rust
fn main() {
    // Iterators เป็น lazy - ไม่ทำงานจนกว่าจะ consume
    
    // นี่ยังไม่ทำอะไร (ไม่มี consumer)
    let lazy_iter = (0..1_000_000)
        .filter(|x| x % 2 == 0)
        .map(|x| x * x);
    
    // ทำงานจริงตอน take(5).collect()
    let first_five: Vec<u64> = lazy_iter.take(5).collect();
    println!("{:?}", first_five);  // [0, 4, 16, 36, 64]

    // เปรียบเทียบ: eager vs lazy
    // Eager (ไม่มีประสิทธิภาพ - สร้าง Vec ชั่วคราว)
    let _eager_result: Vec<i32> = (0..1000)
        .collect::<Vec<_>>()  // สร้าง Vec ทั้งหมด
        .iter()
        .filter(|&&x| x % 2 == 0)
        .copied()
        .collect::<Vec<_>>()
        .iter()
        .map(|&x| x * x)
        .collect();

    // Lazy (มีประสิทธิภาพ - ไม่สร้าง intermediate collections)
    let lazy_result: Vec<i32> = (0..1000)
        .filter(|&x| x % 2 == 0)
        .map(|x| x * x)
        .collect();
    println!("Lazy result count: {}", lazy_result.len());

    // infinite iterator กับ lazy evaluation
    let first_10_squares: Vec<u64> = (0u64..)  // infinite range
        .map(|x| x * x)
        .take(10)
        .collect();
    println!("Squares: {:?}", first_10_squares);

    // scan - เหมือน fold แต่ yield ค่า intermediate
    let running_sum: Vec<i32> = (1..=5)
        .scan(0, |acc, x| {
            *acc += x;
            Some(*acc)
        })
        .collect();
    println!("Running sum: {:?}", running_sum);  // [1, 3, 6, 10, 15]

    // cycle - วนซ้ำ iterator ไปเรื่อยๆ
    let pattern: Vec<i32> = vec![1, 2, 3]
        .into_iter()
        .cycle()
        .take(9)
        .collect();
    println!("Pattern: {:?}", pattern);  // [1, 2, 3, 1, 2, 3, 1, 2, 3]

    // repeat - ทำซ้ำค่าเดิม
    use std::iter;
    let zeros: Vec<i32> = iter::repeat(0).take(5).collect();
    println!("Zeros: {:?}", zeros);  // [0, 0, 0, 0, 0]

    // once - iterator ที่ให้ค่าเดียว
    let one_item: Vec<i32> = iter::once(42).collect();
    println!("{:?}", one_item);  // [42]

    // from_fn - สร้าง iterator จาก closure
    let mut state = 0;
    let counter = iter::from_fn(move || {
        state += 1;
        if state <= 5 { Some(state) } else { None }
    });
    let vals: Vec<i32> = counter.collect();
    println!("Counter: {:?}", vals);  // [1, 2, 3, 4, 5]

    // successors - สร้าง iterator โดย apply function ซ้ำ
    let powers_of_2: Vec<u32> = iter::successors(Some(1u32), |&n| {
        if n < 1000 { Some(n * 2) } else { None }
    }).collect();
    println!("Powers of 2: {:?}", powers_of_2);  // [1, 2, 4, 8, ..., 512]
}
```

---

## 9. Iterator Chaining

```rust
fn main() {
    // Complex chaining example
    let data = vec![
        ("Alice", vec![85, 92, 78, 96]),
        ("Bob", vec![70, 88, 65, 91]),
        ("Charlie", vec![95, 97, 88, 100]),
        ("Diana", vec![60, 72, 80, 68]),
    ];

    // หา students ที่ average score >= 85, เรียงตาม score
    let mut high_scorers: Vec<(&str, f64)> = data.iter()
        .map(|(name, scores)| {
            let avg = scores.iter().sum::<i32>() as f64 / scores.len() as f64;
            (name.as_str(), avg)  // ใช้ as_str() เพื่อได้ &str
        })
        .filter(|(_, avg)| *avg >= 85.0)
        .collect();

    high_scorers.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());

    for (name, avg) in &high_scorers {
        println!("{}: {:.1}", name, avg);
    }

    // Chaining with flat_map
    let sentences = vec![
        "the quick brown fox",
        "jumps over the lazy dog",
    ];
    
    let word_lengths: Vec<(String, usize)> = sentences.iter()
        .flat_map(|s| s.split_whitespace())
        .map(|word| (word.to_uppercase(), word.len()))
        .filter(|(_, len)| *len > 3)
        .collect();
    
    println!("{:?}", word_lengths);

    // Window-style processing
    let prices = vec![100.0, 105.0, 98.0, 112.0, 108.0, 115.0];
    
    let changes: Vec<f64> = prices.windows(2)
        .map(|w| (w[1] - w[0]) / w[0] * 100.0)
        .collect();
    
    println!("Price changes: {:?}", 
        changes.iter().map(|x| format!("{:.1}%", x)).collect::<Vec<_>>());

    // Group consecutive elements
    let data2 = vec![1, 1, 2, 2, 2, 3, 1, 1];
    let mut groups: Vec<(i32, usize)> = Vec::new();
    
    data2.iter().for_each(|&x| {
        match groups.last_mut() {
            Some(last) if last.0 == x => last.1 += 1,
            _ => groups.push((x, 1)),
        }
    });
    
    println!("Groups: {:?}", groups);  // [(1, 2), (2, 3), (3, 1), (1, 2)]
}
```

---

## 10. Practical: Data Processing Pipeline

```rust
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Employee {
    id: u32,
    name: String,
    department: String,
    salary: f64,
    years: u32,
}

impl Employee {
    fn new(id: u32, name: &str, dept: &str, salary: f64, years: u32) -> Self {
        Employee {
            id,
            name: name.to_string(),
            department: dept.to_string(),
            salary,
            years,
        }
    }
}

fn main() {
    let employees = vec![
        Employee::new(1, "Alice", "Engineering", 95000.0, 5),
        Employee::new(2, "Bob", "Engineering", 88000.0, 3),
        Employee::new(3, "Charlie", "Marketing", 72000.0, 7),
        Employee::new(4, "Diana", "Engineering", 105000.0, 8),
        Employee::new(5, "Eve", "Marketing", 68000.0, 2),
        Employee::new(6, "Frank", "HR", 65000.0, 4),
        Employee::new(7, "Grace", "Engineering", 92000.0, 6),
        Employee::new(8, "Henry", "HR", 70000.0, 9),
        Employee::new(9, "Ivy", "Marketing", 75000.0, 5),
        Employee::new(10, "Jack", "Engineering", 110000.0, 10),
    ];

    // 1. Average salary by department
    let dept_salaries: HashMap<&str, Vec<f64>> = employees.iter()
        .fold(HashMap::new(), |mut acc, emp| {
            acc.entry(&emp.department).or_default().push(emp.salary);
            acc
        });

    let mut dept_avg: Vec<(&str, f64)> = dept_salaries.iter()
        .map(|(&dept, salaries)| {
            let avg = salaries.iter().sum::<f64>() / salaries.len() as f64;
            (dept, avg)
        })
        .collect();
    
    dept_avg.sort_by(|a, b| b.1.partial_cmp(&a.1).unwrap());
    
    println!("=== Average Salary by Department ===");
    for (dept, avg) in &dept_avg {
        println!("  {}: ${:.0}", dept, avg);
    }

    // 2. Top 3 highest paid employees
    let mut sorted_employees = employees.clone();
    sorted_employees.sort_by(|a, b| b.salary.partial_cmp(&a.salary).unwrap());
    
    println!("\n=== Top 3 Highest Paid ===");
    sorted_employees.iter()
        .take(3)
        .enumerate()
        .for_each(|(i, emp)| {
            println!("  {}. {} ({}): ${:.0}", i + 1, emp.name, emp.department, emp.salary);
        });

    // 3. Senior employees (5+ years) in Engineering
    let senior_engineers: Vec<&Employee> = employees.iter()
        .filter(|emp| emp.department == "Engineering" && emp.years >= 5)
        .collect();
    
    println!("\n=== Senior Engineers (5+ years) ===");
    senior_engineers.iter()
        .for_each(|emp| println!("  {} ({} years)", emp.name, emp.years));

    // 4. Total payroll and stats
    let total_payroll: f64 = employees.iter().map(|e| e.salary).sum();
    let avg_salary = total_payroll / employees.len() as f64;
    let max_salary = employees.iter().map(|e| e.salary).fold(f64::NEG_INFINITY, f64::max);
    let min_salary = employees.iter().map(|e| e.salary).fold(f64::INFINITY, f64::min);
    
    println!("\n=== Company Statistics ===");
    println!("  Total employees: {}", employees.len());
    println!("  Total payroll: ${:.0}", total_payroll);
    println!("  Average salary: ${:.0}", avg_salary);
    println!("  Max salary: ${:.0}", max_salary);
    println!("  Min salary: ${:.0}", min_salary);

    // 5. Department headcount
    let headcount: HashMap<&str, usize> = employees.iter()
        .fold(HashMap::new(), |mut acc, emp| {
            *acc.entry(emp.department.as_str()).or_insert(0) += 1;
            acc
        });
    
    println!("\n=== Headcount by Department ===");
    let mut hc_sorted: Vec<_> = headcount.iter().collect();
    hc_sorted.sort_by_key(|&(dept, _)| *dept);
    hc_sorted.iter().for_each(|(dept, count)| {
        println!("  {}: {} people", dept, count);
    });

    // 6. Salary raise pipeline (5% raise for 5+ year employees)
    let raised_payroll: f64 = employees.iter()
        .map(|emp| {
            if emp.years >= 5 {
                emp.salary * 1.05
            } else {
                emp.salary
            }
        })
        .sum();
    
    println!("\n=== After 5% Raise for 5+ Years ===");
    println!("  New total payroll: ${:.0}", raised_payroll);
    println!("  Increase: ${:.0}", raised_payroll - total_payroll);

    // 7. Create employee lookup map
    let emp_map: HashMap<u32, &Employee> = employees.iter()
        .map(|emp| (emp.id, emp))
        .collect();
    
    println!("\n=== Employee Lookup ===");
    if let Some(emp) = emp_map.get(&5) {
        println!("  Employee #5: {} in {}", emp.name, emp.department);
    }

    // 8. Employees grouped by salary range
    let salary_groups: HashMap<&str, Vec<&str>> = employees.iter()
        .fold(HashMap::new(), |mut acc, emp| {
            let range = if emp.salary < 70000.0 { "< $70k" }
                        else if emp.salary < 90000.0 { "$70k-$90k" }
                        else if emp.salary < 110000.0 { "$90k-$110k" }
                        else { "> $110k" };
            acc.entry(range).or_default().push(emp.name.as_str());
            acc
        });
    
    println!("\n=== Salary Distribution ===");
    let mut groups_sorted: Vec<_> = salary_groups.iter().collect();
    groups_sorted.sort_by_key(|(range, _)| *range);
    groups_sorted.iter().for_each(|(range, names)| {
        println!("  {}: {:?}", range, names);
    });
}
```

---

## 11. Advanced Iterator Patterns

```rust
use std::collections::HashMap;

// Iterator Combinator Pattern
struct Pipeline<T> {
    data: Vec<T>,
}

impl<T: Clone> Pipeline<T> {
    fn new(data: Vec<T>) -> Self {
        Pipeline { data }
    }

    fn transform<U, F: Fn(T) -> U>(self, f: F) -> Pipeline<U> {
        Pipeline {
            data: self.data.into_iter().map(f).collect(),
        }
    }

    fn filter<F: Fn(&T) -> bool>(self, f: F) -> Pipeline<T> {
        Pipeline {
            data: self.data.into_iter().filter(|x| f(x)).collect(),
        }
    }

    fn result(self) -> Vec<T> {
        self.data
    }
}

fn main() {
    // Using Pipeline combinator
    let result = Pipeline::new(vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
        .filter(|&x| x % 2 == 0)
        .transform(|x| x * x)
        .filter(|&x| x > 10)
        .result();
    
    println!("Pipeline result: {:?}", result);  // [16, 36, 64, 100]

    // Word frequency analysis
    let text = "the quick brown fox jumps over the lazy dog the fox";
    
    let word_freq: HashMap<&str, usize> = text.split_whitespace()
        .fold(HashMap::new(), |mut acc, word| {
            *acc.entry(word).or_insert(0) += 1;
            acc
        });
    
    let mut sorted_words: Vec<(&&str, &usize)> = word_freq.iter().collect();
    sorted_words.sort_by(|a, b| b.1.cmp(a.1).then(a.0.cmp(b.0)));
    
    println!("\nWord frequencies:");
    sorted_words.iter().take(5).for_each(|(word, count)| {
        println!("  '{}': {}", word, count);
    });

    // Matrix transposition with iterators
    let matrix = vec![
        vec![1, 2, 3],
        vec![4, 5, 6],
        vec![7, 8, 9],
    ];
    
    let transposed: Vec<Vec<i32>> = (0..3)
        .map(|col| matrix.iter().map(|row| row[col]).collect())
        .collect();
    
    println!("\nOriginal matrix:");
    matrix.iter().for_each(|row| println!("  {:?}", row));
    println!("Transposed:");
    transposed.iter().for_each(|row| println!("  {:?}", row));

    // Sliding window average
    let data = vec![1.0, 3.0, 5.0, 7.0, 9.0, 11.0, 13.0];
    let window_size = 3;
    
    let moving_avg: Vec<f64> = data.windows(window_size)
        .map(|w| w.iter().sum::<f64>() / window_size as f64)
        .collect();
    
    println!("\nMoving average (window=3): {:?}", moving_avg);
    // [3.0, 5.0, 7.0, 9.0, 11.0]

    // Cartesian product
    let colors = vec!["red", "blue", "green"];
    let sizes = vec!["S", "M", "L"];
    
    let products: Vec<(&str, &str)> = colors.iter()
        .flat_map(|&c| sizes.iter().map(move |&s| (c, s)))
        .collect();
    
    println!("\nProduct variants: {} items", products.len());
    products.iter().take(4).for_each(|(c, s)| print!("{}/{} ", c, s));
    println!("...");
}
```

---

## 12. สรุป

| Feature | Description | ตัวอย่าง |
|---------|-------------|---------|
| Closure | Anonymous function | `\|x\| x + 1` |
| Capture by ref | ใช้ค่าจาก scope | `let f = \|n\| n + x;` |
| move closure | เป็นเจ้าของ captured values | `move \|\| println!("{}", name)` |
| Fn | เรียกซ้ำได้ ไม่เปลี่ยนค่า | `fn call<F: Fn()>(f: F)` |
| FnMut | เรียกซ้ำได้ เปลี่ยนค่าได้ | `fn call<F: FnMut()>(mut f: F)` |
| FnOnce | เรียกครั้งเดียว | `fn call<F: FnOnce()>(f: F)` |
| Iterator | Lazy sequence | `iter.map().filter()` |
| Adapters | แปลง iterator | `map, filter, flat_map, zip` |
| Consumers | บริโภค iterator | `collect, fold, sum, any, all` |

### Key Points ที่ต้องจำ

1. **Closures capture by reference โดย default** - ใช้ `move` เพื่อ ownership
2. **Iterators เป็น lazy** - ไม่ทำงานจนกว่าจะมี consumer
3. **Fn > FnMut > FnOnce** - Fn เข้มงวดที่สุด, FnOnce หลวมที่สุด
4. **Iterator chaining** สร้าง zero-cost abstractions
5. **collect()** ต้องบอก type เสมอ (หรือ type inference ช่วยได้)

---

*[← Part 010: Lifetimes และ Memory Safety](../part_010/README.md) | [Part 012: Modules, Crates, และ Packages →](../part_012/README.md)*
