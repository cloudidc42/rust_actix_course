# Part 007: Collections: Vec, HashMap, HashSet 📦

## 🎯 เป้าหมายของ Part นี้

- ใช้ Vec<T> อย่างเต็มประสิทธิภาพ
- ใช้ HashMap<K, V>
- ใช้ HashSet<T>
- รู้จัก BTreeMap, BTreeSet, VecDeque
- Performance considerations

---

## 1. Vec<T>

### 1.1 Vec พื้นฐาน

```rust
fn main() {
    // สร้าง Vec
    let mut v1: Vec<i32> = Vec::new();
    let mut v2 = vec![1, 2, 3, 4, 5];
    let v3: Vec<i32> = (1..=10).collect();
    let v4 = vec![0; 5];  // [0, 0, 0, 0, 0]

    // push / pop
    v1.push(10);
    v1.push(20);
    v1.push(30);
    println!("{:?}", v1);
    println!("Popped: {:?}", v1.pop());  // Some(30)

    // Insert / remove
    v2.insert(2, 99);       // Insert 99 at index 2
    println!("After insert: {:?}", v2);  // [1, 2, 99, 3, 4, 5]
    v2.remove(2);           // Remove at index 2
    println!("After remove: {:?}", v2);  // [1, 2, 3, 4, 5]

    // Access
    let first = v2[0];
    let safe = v2.get(100);  // Option<&T>
    println!("first: {}, safe get: {:?}", first, safe);

    // Slices
    let slice = &v2[1..4];
    println!("slice: {:?}", slice);

    // Length
    println!("len: {}, is_empty: {}", v2.len(), v2.is_empty());

    // Sorting
    let mut nums = vec![3, 1, 4, 1, 5, 9, 2, 6, 5, 3];
    nums.sort();
    println!("sorted: {:?}", nums);

    nums.sort_by(|a, b| b.cmp(a));  // reverse sort
    println!("reverse sorted: {:?}", nums);

    // Dedup (remove consecutive duplicates - must sort first)
    nums.sort();
    nums.dedup();
    println!("deduped: {:?}", nums);

    // Retain (filter in place)
    nums.retain(|&x| x % 2 == 0);
    println!("evens only: {:?}", nums);

    // Extend
    let mut a = vec![1, 2, 3];
    let b = vec![4, 5, 6];
    a.extend(&b);
    println!("extended: {:?}", a);

    // Drain
    let mut source = vec![1, 2, 3, 4, 5];
    let drained: Vec<i32> = source.drain(1..3).collect();
    println!("drained: {:?}, remaining: {:?}", drained, source);

    // Flatten
    let nested = vec![vec![1, 2], vec![3, 4], vec![5, 6]];
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("flat: {:?}", flat);

    // Windows and chunks
    let data = vec![1, 2, 3, 4, 5, 6];
    print!("windows(3): ");
    for w in data.windows(3) {
        print!("{:?} ", w);
    }
    println!();

    print!("chunks(2): ");
    for c in data.chunks(2) {
        print!("{:?} ", c);
    }
    println!();
}
```

### 1.2 Vec กับ Structs

```rust
#[derive(Debug, Clone)]
struct Student {
    name: String,
    score: f64,
}

impl Student {
    fn new(name: &str, score: f64) -> Self {
        Self { name: name.to_string(), score }
    }
}

fn main() {
    let mut students = vec![
        Student::new("Alice", 85.5),
        Student::new("Bob", 92.0),
        Student::new("Charlie", 78.0),
        Student::new("Diana", 95.5),
        Student::new("Eve", 88.0),
    ];

    // Sort by score
    students.sort_by(|a, b| b.score.partial_cmp(&a.score).unwrap());
    println!("Ranked students:");
    for (i, s) in students.iter().enumerate() {
        println!("  {}. {} - {:.1}", i + 1, s.name, s.score);
    }

    // Average
    let avg = students.iter().map(|s| s.score).sum::<f64>() / students.len() as f64;
    println!("Average: {:.1}", avg);

    // Filter passing (>= 80)
    let passing: Vec<&Student> = students.iter().filter(|s| s.score >= 80.0).collect();
    println!("Passing: {:?}", passing.iter().map(|s| &s.name).collect::<Vec<_>>());

    // Find top scorer
    if let Some(top) = students.first() {
        println!("Top scorer: {} ({:.1})", top.name, top.score);
    }

    // Group by pass/fail
    let (pass, fail): (Vec<_>, Vec<_>) = students.iter().partition(|s| s.score >= 80.0);
    println!("Pass: {}, Fail: {}", pass.len(), fail.len());
}
```

---

## 2. HashMap<K, V>

### 2.1 HashMap พื้นฐาน

```rust
use std::collections::HashMap;

fn main() {
    // สร้าง HashMap
    let mut scores: HashMap<String, i32> = HashMap::new();

    // Insert
    scores.insert(String::from("Alice"), 95);
    scores.insert(String::from("Bob"), 87);
    scores.insert(String::from("Charlie"), 92);

    // ด้วย vec ของ tuples
    let teams: HashMap<_, _> = vec![
        ("Blues", 10),
        ("Reds", 50),
        ("Greens", 30),
    ].into_iter().collect();

    // Access
    let alice_score = &scores["Alice"];  // panics if not found
    let bob_score = scores.get("Bob");   // Option<&V>
    let missing = scores.get("Unknown");

    println!("Alice: {}", alice_score);
    println!("Bob: {:?}", bob_score);
    println!("Unknown: {:?}", missing);

    // entry API (insert if not exists)
    scores.entry(String::from("Dave")).or_insert(80);
    scores.entry(String::from("Alice")).or_insert(0);  // ไม่เปลี่ยน ถ้ามีอยู่แล้ว
    println!("Dave: {}", scores["Dave"]);
    println!("Alice still: {}", scores["Alice"]);

    // Update via entry
    let text = "hello world wonderful world hello";
    let mut word_count: HashMap<&str, u32> = HashMap::new();
    for word in text.split_whitespace() {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }
    println!("Word counts: {:?}", word_count);

    // Iterate
    println!("\nAll scores:");
    for (name, score) in &scores {
        println!("  {}: {}", name, score);
    }

    // Check existence
    println!("Has Alice: {}", scores.contains_key("Alice"));
    println!("Has Eve: {}", scores.contains_key("Eve"));

    // Remove
    let removed = scores.remove("Charlie");
    println!("Removed Charlie: {:?}", removed);

    // Length
    println!("Remaining: {}", scores.len());

    // Keys and values
    let keys: Vec<&String> = scores.keys().collect();
    let values: Vec<&i32> = scores.values().collect();
    println!("Keys: {:?}", keys);
    println!("Values: {:?}", values);

    // Max/min
    let max_score = scores.values().max();
    let max_name = scores.iter().max_by_key(|(_, &v)| v);
    println!("Max score: {:?}", max_score);
    println!("Top scorer: {:?}", max_name);
}
```

### 2.2 HashMap ขั้นสูง

```rust
use std::collections::HashMap;

fn group_by<T, K, F>(items: Vec<T>, key_fn: F) -> HashMap<K, Vec<T>>
where
    K: std::hash::Hash + Eq,
    F: Fn(&T) -> K,
{
    let mut map: HashMap<K, Vec<T>> = HashMap::new();
    for item in items {
        map.entry(key_fn(&item)).or_insert_with(Vec::new).push(item);
    }
    map
}

#[derive(Debug)]
struct Person {
    name: String,
    department: String,
    salary: f64,
}

fn main() {
    let employees = vec![
        Person { name: "Alice".to_string(), department: "Engineering".to_string(), salary: 80000.0 },
        Person { name: "Bob".to_string(), department: "Marketing".to_string(), salary: 60000.0 },
        Person { name: "Charlie".to_string(), department: "Engineering".to_string(), salary: 90000.0 },
        Person { name: "Diana".to_string(), department: "HR".to_string(), salary: 55000.0 },
        Person { name: "Eve".to_string(), department: "Engineering".to_string(), salary: 85000.0 },
        Person { name: "Frank".to_string(), department: "Marketing".to_string(), salary: 65000.0 },
    ];

    // Group by department
    let by_dept = group_by(employees, |p| p.department.clone());

    for (dept, people) in &by_dept {
        let avg_salary = people.iter().map(|p| p.salary).sum::<f64>() / people.len() as f64;
        println!("{}: {} employees, avg salary: ${:.0}",
            dept, people.len(), avg_salary);
        for p in people {
            println!("  - {} (${:.0})", p.name, p.salary);
        }
    }

    // Two-level HashMap
    let mut matrix: HashMap<i32, HashMap<i32, i32>> = HashMap::new();
    for i in 1..=3 {
        for j in 1..=3 {
            matrix.entry(i).or_insert_with(HashMap::new).insert(j, i * j);
        }
    }
    println!("\nMultiplication table:");
    for i in 1..=3 {
        for j in 1..=3 {
            print!("{:3}", matrix[&i][&j]);
        }
        println!();
    }
}
```

---

## 3. HashSet<T>

### 3.1 HashSet พื้นฐาน

```rust
use std::collections::HashSet;

fn main() {
    // สร้าง HashSet
    let mut set1: HashSet<i32> = HashSet::new();
    let set2: HashSet<i32> = [1, 2, 3, 4, 5].into_iter().collect();
    let set3: HashSet<i32> = vec![3, 4, 5, 6, 7].into_iter().collect();

    // Insert
    set1.insert(1);
    set1.insert(2);
    set1.insert(2);  // duplicate ไม่ถูกเพิ่ม
    set1.insert(3);
    println!("set1: {:?}", set1);  // ไม่มี duplicate

    // Contains
    println!("Contains 2: {}", set1.contains(&2));
    println!("Contains 5: {}", set1.contains(&5));

    // Remove
    set1.remove(&2);
    println!("After remove 2: {:?}", set1);

    // Set operations
    let union: HashSet<&i32> = set2.union(&set3).collect();
    let intersection: HashSet<&i32> = set2.intersection(&set3).collect();
    let difference: HashSet<&i32> = set2.difference(&set3).collect();
    let sym_diff: HashSet<&i32> = set2.symmetric_difference(&set3).collect();

    println!("\nset2: {:?}", set2);
    println!("set3: {:?}", set3);
    println!("union: {:?}", union);
    println!("intersection: {:?}", intersection);
    println!("difference (set2 - set3): {:?}", difference);
    println!("symmetric difference: {:?}", sym_diff);

    // Subset/Superset
    let small: HashSet<i32> = [1, 2].into_iter().collect();
    let large: HashSet<i32> = [1, 2, 3, 4].into_iter().collect();
    println!("small is subset of large: {}", small.is_subset(&large));
    println!("large is superset of small: {}", large.is_superset(&small));

    // Remove duplicates from Vec
    let numbers = vec![1, 2, 3, 2, 1, 4, 3, 5];
    let unique: HashSet<_> = numbers.iter().cloned().collect();
    let mut unique_vec: Vec<_> = unique.into_iter().collect();
    unique_vec.sort();
    println!("Unique (sorted): {:?}", unique_vec);
}
```

---

## 4. Other Collections

### 4.1 BTreeMap (sorted)

```rust
use std::collections::BTreeMap;

fn main() {
    // BTreeMap - sorted by key
    let mut map = BTreeMap::new();
    map.insert("banana", 3);
    map.insert("apple", 5);
    map.insert("cherry", 1);
    map.insert("date", 2);

    // Iterates in sorted order!
    println!("Sorted fruits:");
    for (fruit, count) in &map {
        println!("  {}: {}", fruit, count);
    }

    // Range queries
    let range: BTreeMap<_, _> = map.range("apple"..="cherry").collect();
    println!("Range apple..=cherry: {:?}", range);

    // First/Last
    println!("First: {:?}", map.iter().next());
    println!("Last: {:?}", map.iter().next_back());
}
```

### 4.2 VecDeque (double-ended queue)

```rust
use std::collections::VecDeque;

fn main() {
    let mut deque: VecDeque<i32> = VecDeque::new();

    // Push front and back
    deque.push_back(3);
    deque.push_back(4);
    deque.push_front(2);
    deque.push_front(1);
    println!("{:?}", deque);  // [1, 2, 3, 4]

    // Pop front and back
    println!("Front: {:?}", deque.pop_front());  // Some(1)
    println!("Back: {:?}", deque.pop_back());    // Some(4)
    println!("{:?}", deque);  // [2, 3]

    // BFS queue simulation
    let mut queue: VecDeque<String> = VecDeque::new();
    queue.push_back("task1".to_string());
    queue.push_back("task2".to_string());
    queue.push_back("task3".to_string());

    while let Some(task) = queue.pop_front() {
        println!("Processing: {}", task);
    }
}
```

---

## 5. สรุปและ Exercises

### 5.1 สิ่งที่เรียนรู้

✅ Vec: push, pop, insert, remove, sort, filter  
✅ HashMap: insert, get, entry, iterate  
✅ HashSet: set operations (union, intersection, etc.)  
✅ BTreeMap: sorted map  
✅ VecDeque: double-ended queue  

### 5.2 Exercise

**Exercise: Inventory System**
```rust
// สร้าง inventory system ที่ track:
// - Products (HashMap<String, Product>)
// - Categories (HashSet<String>)
// - ค้นหาสินค้าตาม category, price range
// - Statistics (total value, most stocked, etc.)
```

---

*[← Part 006: Pattern Matching](../part_006/README.md) | [Part 008: Error Handling →](../part_008/README.md)*
