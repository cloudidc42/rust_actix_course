# Part 092: Memory Management Deep Dive

## บทนำ (Introduction)

การจัดการหน่วยความจำเป็นหัวใจสำคัญของ Rust ในบทนี้เราจะเจาะลึกเรื่อง Stack vs Heap, Allocator API, Custom allocators, Memory layout และเทคนิคต่างๆ ในการเขียนโค้ดที่มีประสิทธิภาพด้านหน่วยความจำ

## 1. Stack vs Heap Allocation

### Stack allocation - เร็วและอัตโนมัติ

```rust
fn stack_examples() {
    // ทุกอย่างที่ประกาศในฟังก์ชันอยู่บน Stack
    let x: i32 = 42;              // 4 bytes บน stack
    let y: f64 = 3.14;            // 8 bytes บน stack
    let arr: [i32; 100] = [0; 100]; // 400 bytes บน stack
    
    // Struct ที่ไม่มี heap allocation
    #[derive(Debug)]
    struct Point {
        x: f32,
        y: f32,
    }
    
    let p = Point { x: 1.0, y: 2.0 }; // 8 bytes บน stack
    
    println!("x = {}, y = {}, arr[0] = {}, p = {:?}", x, y, arr[0], p);
    
    // ขนาดของ types
    println!("Size of i32: {} bytes", std::mem::size_of::<i32>());
    println!("Size of f64: {} bytes", std::mem::size_of::<f64>());
    println!("Size of Point: {} bytes", std::mem::size_of::<Point>());
    println!("Size of [i32; 100]: {} bytes", std::mem::size_of::<[i32; 100]>());
}

// Heap allocation ด้วย Box
fn heap_examples() {
    // Box<T> allocates T on the heap
    let boxed_int: Box<i32> = Box::new(42);
    let boxed_arr: Box<[i32; 1000]> = Box::new([0; 1000]); // array ใหญ่ควรอยู่บน heap
    
    println!("Boxed int: {}", boxed_int);
    println!("Boxed array size: {} bytes", std::mem::size_of::<[i32; 1000]>());
    
    // String อยู่บน heap
    let s = String::from("Hello, World!"); // heap allocation
    let s_ref = s.as_str(); // reference (stack) ชี้ไปที่ heap data
    
    println!("String: {}", s);
    println!("Str ref: {}", s_ref);
    
    // Vec<T> อยู่บน heap
    let v: Vec<i32> = vec![1, 2, 3, 4, 5];
    println!("Vec capacity: {}, len: {}", v.capacity(), v.len());
}

fn main() {
    stack_examples();
    heap_examples();
}
```

## 2. Memory Layout of Types

### ตรวจสอบ memory layout

```rust
use std::mem;

#[derive(Debug)]
struct LayoutExample {
    a: u8,   // 1 byte
    b: u32,  // 4 bytes (aligned to 4)
    c: u8,   // 1 byte
    d: u64,  // 8 bytes (aligned to 8)
}

#[derive(Debug)]
#[repr(C)]  // C-compatible layout
struct CLayout {
    a: u8,
    b: u32,
    c: u8,
    d: u64,
}

#[derive(Debug)]
#[repr(packed)]  // no padding
struct PackedLayout {
    a: u8,
    b: u32,
    c: u8,
    d: u64,
}

#[derive(Debug)]
#[repr(align(16))]  // align to 16 bytes
struct AlignedLayout {
    a: u8,
    b: u32,
}

fn main() {
    println!("=== Memory Layout ===");
    println!("Default layout:");
    println!("  Size: {}", mem::size_of::<LayoutExample>());
    println!("  Align: {}", mem::align_of::<LayoutExample>());
    
    println!("\nC layout:");
    println!("  Size: {}", mem::size_of::<CLayout>());
    println!("  Align: {}", mem::align_of::<CLayout>());
    
    println!("\nPacked layout:");
    println!("  Size: {}", mem::size_of::<PackedLayout>());
    println!("  Align: {}", mem::align_of::<PackedLayout>());
    
    println!("\nAligned layout:");
    println!("  Size: {}", mem::size_of::<AlignedLayout>());
    println!("  Align: {}", mem::align_of::<AlignedLayout>());
    
    // ดู offset ของ fields
    let example = LayoutExample {
        a: 1, b: 2, c: 3, d: 4
    };
    let base = &example as *const _ as usize;
    let a_offset = &example.a as *const _ as usize - base;
    let b_offset = &example.b as *const _ as usize - base;
    let c_offset = &example.c as *const _ as usize - base;
    let d_offset = &example.d as *const _ as usize - base;
    
    println!("\nField offsets:");
    println!("  a: +{}", a_offset);
    println!("  b: +{}", b_offset);
    println!("  c: +{}", c_offset);
    println!("  d: +{}", d_offset);
}
```

### Enum memory layout

```rust
use std::mem;

#[derive(Debug)]
enum SimpleEnum {
    A,
    B,
    C,
}

#[derive(Debug)]
enum ComplexEnum {
    None,
    Integer(i64),
    Float(f64),
    Both(i32, i32),
    Named { x: f32, y: f32 },
}

fn enum_layout_demo() {
    println!("SimpleEnum size: {}", mem::size_of::<SimpleEnum>());
    println!("ComplexEnum size: {}", mem::size_of::<ComplexEnum>());
    
    // Option optimization - None เป็น null pointer optimization
    println!("Option<Box<i32>> size: {}", mem::size_of::<Option<Box<i32>>>());
    println!("Box<i32> size: {}", mem::size_of::<Box<i32>>());
    // ทั้งสองขนาดเท่ากัน! เพราะ Rust ใช้ null pointer optimization
    
    println!("Option<i32> size: {}", mem::size_of::<Option<i32>>());
    println!("i32 size: {}", mem::size_of::<i32>());
    // Option<i32> ใหญ่กว่า i32 เพราะต้องเก็บ discriminant
}
```

## 3. Allocator API

### Global allocator

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::sync::atomic::{AtomicUsize, Ordering};

// Custom allocator ที่นับ allocations
struct TrackingAllocator {
    allocations: AtomicUsize,
    total_bytes: AtomicUsize,
}

unsafe impl GlobalAlloc for TrackingAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let ptr = System.alloc(layout);
        if !ptr.is_null() {
            self.allocations.fetch_add(1, Ordering::SeqCst);
            self.total_bytes.fetch_add(layout.size(), Ordering::SeqCst);
        }
        ptr
    }
    
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        System.dealloc(ptr, layout);
        self.allocations.fetch_sub(1, Ordering::SeqCst);
        self.total_bytes.fetch_sub(layout.size(), Ordering::SeqCst);
    }
}

// ใช้ #[global_allocator] attribute เพื่อ override default allocator
// #[global_allocator]
// static ALLOCATOR: TrackingAllocator = TrackingAllocator {
//     allocations: AtomicUsize::new(0),
//     total_bytes: AtomicUsize::new(0),
// };

fn allocator_demo() {
    use std::alloc::{alloc, dealloc, Layout};
    
    unsafe {
        // Manual memory allocation
        let layout = Layout::new::<i32>();
        let ptr = alloc(layout) as *mut i32;
        
        if ptr.is_null() {
            panic!("Allocation failed!");
        }
        
        // Write to allocated memory
        *ptr = 42;
        println!("Allocated i32 at {:?}, value: {}", ptr, *ptr);
        
        // Must deallocate manually
        dealloc(ptr as *mut u8, layout);
        println!("Deallocated");
        
        // Allocate array
        let arr_layout = Layout::array::<i32>(10).unwrap();
        let arr_ptr = alloc(arr_layout) as *mut i32;
        
        for i in 0..10 {
            *arr_ptr.add(i) = i as i32 * i as i32;
        }
        
        for i in 0..10 {
            print!("{} ", *arr_ptr.add(i));
        }
        println!();
        
        dealloc(arr_ptr as *mut u8, arr_layout);
    }
}

fn main() {
    allocator_demo();
}
```

## 4. Custom Allocators (jemalloc)

### ใช้ jemalloc สำหรับ performance

```rust
// Cargo.toml:
// [dependencies]
// tikv-jemallocator = "0.5"
//
// [features]
// jemalloc = ["tikv-jemallocator"]

// ตัวอย่างการใช้งาน (ต้องเพิ่ม dependency)
// use tikv_jemallocator::Jemalloc;
// 
// #[global_allocator]
// static GLOBAL: Jemalloc = Jemalloc;

// Bump allocator - เร็วมาก สำหรับ short-lived allocations
use std::alloc::Layout;
use std::cell::UnsafeCell;

struct BumpAllocator {
    data: UnsafeCell<Vec<u8>>,
    offset: UnsafeCell<usize>,
}

impl BumpAllocator {
    fn new(capacity: usize) -> Self {
        BumpAllocator {
            data: UnsafeCell::new(vec![0u8; capacity]),
            offset: UnsafeCell::new(0),
        }
    }
    
    fn alloc_raw(&self, size: usize, align: usize) -> Option<*mut u8> {
        unsafe {
            let offset = *self.offset.get();
            let data = &mut *self.data.get();
            let ptr = data.as_mut_ptr().add(offset);
            let aligned_offset = (offset + align - 1) & !(align - 1);
            let new_offset = aligned_offset + size;
            
            if new_offset <= data.len() {
                *self.offset.get() = new_offset;
                Some(data.as_mut_ptr().add(aligned_offset))
            } else {
                None
            }
        }
    }
    
    fn reset(&self) {
        unsafe {
            *self.offset.get() = 0;
        }
    }
    
    fn used(&self) -> usize {
        unsafe { *self.offset.get() }
    }
}

fn bump_allocator_demo() {
    let allocator = BumpAllocator::new(1024 * 1024); // 1MB
    
    // Allocate various types
    let int_ptr = allocator.alloc_raw(4, 4).expect("Alloc failed");
    unsafe {
        *(int_ptr as *mut i32) = 42;
        println!("Allocated i32: {}", *(int_ptr as *const i32));
    }
    
    println!("Used: {} bytes", allocator.used());
    
    // Reset = ลบทุกอย่างในทีเดียว (เร็วมาก)
    allocator.reset();
    println!("After reset, used: {} bytes", allocator.used());
}

fn main() {
    bump_allocator_demo();
}
```

## 5. Zero-cost Abstractions

### Iterators เป็น zero-cost

```rust
fn zero_cost_demo() {
    let data: Vec<i32> = (0..1_000_000).collect();
    
    // Iterator chain - compiles to same machine code as imperative loop
    let sum_iter: i64 = data.iter()
        .filter(|&&x| x % 2 == 0)
        .map(|&x| x as i64 * x as i64)
        .sum();
    
    // Equivalent imperative code
    let sum_imperative: i64 = {
        let mut sum = 0i64;
        for &x in &data {
            if x % 2 == 0 {
                sum += x as i64 * x as i64;
            }
        }
        sum
    };
    
    assert_eq!(sum_iter, sum_imperative);
    println!("Both approaches give: {}", sum_iter);
}

// Newtype pattern - zero overhead
struct Meters(f64);
struct Kilograms(f64);

impl Meters {
    fn to_feet(&self) -> f64 {
        self.0 * 3.28084
    }
}

impl std::fmt::Display for Meters {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "{}m", self.0)
    }
}

// Generic abstractions - monomorphized at compile time
fn sum_generic<T: std::iter::Sum + Copy>(slice: &[T]) -> T {
    slice.iter().copied().sum()
}

fn main() {
    zero_cost_demo();
    
    let m = Meters(1.0);
    println!("{} = {} feet", m, m.to_feet());
    
    let ints = [1, 2, 3, 4, 5];
    let floats = [1.0f64, 2.0, 3.0, 4.0, 5.0];
    
    println!("Int sum: {}", sum_generic(&ints));
    println!("Float sum: {}", sum_generic(&floats));
}
```

## 6. Avoiding Memory Leaks

### Memory leak patterns และวิธีหลีกเลี่ยง

```rust
use std::rc::Rc;
use std::cell::RefCell;

// Reference cycle สร้าง memory leak!
#[derive(Debug)]
struct Node {
    value: i32,
    next: Option<Rc<RefCell<Node>>>,
}

fn create_cycle() {
    let node1 = Rc::new(RefCell::new(Node { value: 1, next: None }));
    let node2 = Rc::new(RefCell::new(Node { value: 2, next: None }));
    
    // สร้าง cycle - LEAK!
    node1.borrow_mut().next = Some(Rc::clone(&node2));
    node2.borrow_mut().next = Some(Rc::clone(&node1)); // cycle!
    
    // เมื่อ node1 และ node2 out of scope
    // Rc::drop ลด count แต่ยังไม่ถึง 0 เพราะ cycle
    // -> memory leak!
    println!("node1 ref count: {}", Rc::strong_count(&node1)); // 2
    println!("node2 ref count: {}", Rc::strong_count(&node2)); // 2
}

// แก้ด้วย Weak references
use std::rc::Weak;

#[derive(Debug)]
struct SafeNode {
    value: i32,
    next: Option<Rc<RefCell<SafeNode>>>,
    prev: Option<Weak<RefCell<SafeNode>>>, // Weak prevents cycle
}

fn safe_linked_list() {
    let node1 = Rc::new(RefCell::new(SafeNode {
        value: 1,
        next: None,
        prev: None,
    }));
    
    let node2 = Rc::new(RefCell::new(SafeNode {
        value: 2,
        next: None,
        prev: Some(Rc::downgrade(&node1)), // Weak reference
    }));
    
    node1.borrow_mut().next = Some(Rc::clone(&node2));
    
    println!("node1 strong: {}", Rc::strong_count(&node1)); // 1
    println!("node2 strong: {}", Rc::strong_count(&node2)); // 2
    println!("node2 weak: {}", Rc::weak_count(&node2)); // 0
    
    // เมื่อ node1, node2 drop -> memory freed properly
}

// Drop order matters
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Dropping resource: {}", self.name);
    }
}

fn drop_order_demo() {
    // ลำดับการ drop เป็น LIFO (reverse declaration order)
    let r1 = Resource { name: "first".to_string() };
    let r2 = Resource { name: "second".to_string() };
    let r3 = Resource { name: "third".to_string() };
    
    println!("Resources created");
    // r3 dropped first, then r2, then r1
}

fn main() {
    create_cycle();
    safe_linked_list();
    drop_order_demo();
}
```

## 7. Memory-efficient Data Structures

### SmallVec และ techniques อื่นๆ

```rust
// Small buffer optimization
struct SmallVec<T, const N: usize> {
    data: SmallVecData<T, N>,
    len: usize,
}

enum SmallVecData<T, const N: usize> {
    Inline([std::mem::MaybeUninit<T>; N]),
    Heap(Vec<T>),
}

// Compact types
use std::num::NonZeroU32;

// Option<NonZeroU32> ขนาดเท่ากับ u32 (null pointer optimization)
fn compact_types() {
    println!("u32 size: {}", std::mem::size_of::<u32>());
    println!("NonZeroU32 size: {}", std::mem::size_of::<NonZeroU32>());
    println!("Option<NonZeroU32> size: {}", std::mem::size_of::<Option<NonZeroU32>>());
    // เท่ากัน!
    
    println!("Option<u32> size: {}", std::mem::size_of::<Option<u32>>());
    // ใหญ่กว่า u32
}

// Arena allocation pattern
struct Arena {
    chunks: Vec<Vec<u8>>,
    current_chunk: Vec<u8>,
    chunk_size: usize,
}

impl Arena {
    fn new(chunk_size: usize) -> Self {
        Arena {
            chunks: Vec::new(),
            current_chunk: Vec::with_capacity(chunk_size),
            chunk_size,
        }
    }
    
    fn alloc_slice(&mut self, size: usize) -> &mut [u8] {
        if self.current_chunk.len() + size > self.chunk_size {
            let old_chunk = std::mem::replace(
                &mut self.current_chunk,
                Vec::with_capacity(self.chunk_size)
            );
            self.chunks.push(old_chunk);
        }
        
        let start = self.current_chunk.len();
        self.current_chunk.extend(std::iter::repeat(0).take(size));
        &mut self.current_chunk[start..]
    }
}

// String interning - ลด memory ด้วยการ share strings
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

struct StringInterner {
    pool: HashMap<String, Arc<str>>,
}

impl StringInterner {
    fn new() -> Self {
        StringInterner {
            pool: HashMap::new(),
        }
    }
    
    fn intern(&mut self, s: &str) -> Arc<str> {
        if let Some(interned) = self.pool.get(s) {
            Arc::clone(interned)
        } else {
            let arc: Arc<str> = Arc::from(s);
            self.pool.insert(s.to_string(), Arc::clone(&arc));
            arc
        }
    }
}

fn string_interning_demo() {
    let mut interner = StringInterner::new();
    
    let s1 = interner.intern("hello");
    let s2 = interner.intern("hello");
    let s3 = interner.intern("world");
    
    // s1 และ s2 point ไปที่ memory เดียวกัน
    println!("s1 ptr: {:p}", s1.as_ptr());
    println!("s2 ptr: {:p}", s2.as_ptr());
    println!("s3 ptr: {:p}", s3.as_ptr());
    println!("s1 == s2 ptr: {}", Arc::ptr_eq(&s1, &s2)); // true
}

fn main() {
    compact_types();
    string_interning_demo();
    
    let mut arena = Arena::new(1024);
    let slice = arena.alloc_slice(16);
    slice[0] = 42;
    println!("Arena allocated {} bytes at index 0: {}", slice.len(), slice[0]);
}
```

## 8. Practical: Memory-efficient Data Structures

### Cache-friendly data structures

```rust
use std::collections::HashMap;

// AoS (Array of Structs) vs SoA (Struct of Arrays)
// SoA ดีกว่าสำหรับ SIMD และ cache performance

// AoS - ไม่ดีสำหรับ cache ถ้าเราเข้าถึง field เดียว
#[derive(Debug, Clone)]
struct ParticleAoS {
    x: f32,
    y: f32,
    z: f32,
    velocity_x: f32,
    velocity_y: f32,
    velocity_z: f32,
    mass: f32,
}

// SoA - ดีสำหรับ cache ถ้าประมวลผล field เดียวกันทุก particle
struct ParticlesSoA {
    x: Vec<f32>,
    y: Vec<f32>,
    z: Vec<f32>,
    velocity_x: Vec<f32>,
    velocity_y: Vec<f32>,
    velocity_z: Vec<f32>,
    mass: Vec<f32>,
}

impl ParticlesSoA {
    fn new(count: usize) -> Self {
        ParticlesSoA {
            x: vec![0.0; count],
            y: vec![0.0; count],
            z: vec![0.0; count],
            velocity_x: vec![0.0; count],
            velocity_y: vec![0.0; count],
            velocity_z: vec![0.0; count],
            mass: vec![1.0; count],
        }
    }
    
    fn update_positions(&mut self, dt: f32) {
        // ทำ SIMD-friendly loop บน contiguous memory
        for i in 0..self.x.len() {
            self.x[i] += self.velocity_x[i] * dt;
            self.y[i] += self.velocity_y[i] * dt;
            self.z[i] += self.velocity_z[i] * dt;
        }
    }
    
    fn count(&self) -> usize {
        self.x.len()
    }
}

// Memory pool สำหรับ frequent alloc/dealloc
struct ObjectPool<T> {
    available: Vec<T>,
    create: Box<dyn Fn() -> T>,
}

impl<T> ObjectPool<T> {
    fn new(initial_size: usize, create: impl Fn() -> T + 'static) -> Self {
        let create = Box::new(create);
        let available = (0..initial_size).map(|_| create()).collect();
        ObjectPool { available, create }
    }
    
    fn acquire(&mut self) -> T {
        self.available.pop().unwrap_or_else(|| (self.create)())
    }
    
    fn release(&mut self, obj: T) {
        self.available.push(obj);
    }
    
    fn pool_size(&self) -> usize {
        self.available.len()
    }
}

fn memory_benchmark() {
    use std::time::Instant;
    
    let n = 100_000;
    
    // Test SoA
    let mut particles = ParticlesSoA::new(n);
    for i in 0..n {
        particles.velocity_x[i] = (i as f32).sin();
        particles.velocity_y[i] = (i as f32).cos();
    }
    
    let start = Instant::now();
    for _ in 0..100 {
        particles.update_positions(0.016);
    }
    println!("SoA update {} particles x100: {:?}", n, start.elapsed());
    
    // Object pool
    let mut pool: ObjectPool<Vec<u8>> = ObjectPool::new(10, || Vec::with_capacity(1024));
    
    println!("Pool size before: {}", pool.pool_size());
    let mut buffers: Vec<Vec<u8>> = (0..5).map(|_| pool.acquire()).collect();
    println!("Pool size after acquire 5: {}", pool.pool_size());
    
    for buf in buffers.drain(..) {
        pool.release(buf);
    }
    println!("Pool size after release: {}", pool.pool_size());
}

fn main() {
    memory_benchmark();
}
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **Stack vs Heap**: ความแตกต่างและเมื่อไรควรใช้อะไร
- **Memory Layout**: padding, alignment, repr attributes
- **Allocator API**: manual allocation และ custom allocators
- **Zero-cost Abstractions**: iterators ไม่มี overhead
- **Memory Leaks**: reference cycles และการใช้ Weak references
- **Efficient Structures**: Arena allocation, string interning, SoA pattern
- **Object Pool**: reuse objects เพื่อลด allocation overhead

---

[← Part 091](../part_091/README.md) | [Part 093 →](../part_093/README.md)
