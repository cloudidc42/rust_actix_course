# Part 093: Unsafe Rust

## บทนำ (Introduction)

`unsafe` ใน Rust คือ escape hatch ที่ช่วยให้เราทำสิ่งที่ Rust's borrow checker ปกติห้าม เราจะเรียนรู้ว่าเมื่อไรควรใช้ unsafe, วิธีใช้ raw pointers, FFI, และวิธีเขียนโค้ดที่ปลอดภัยโดยใช้ unsafe ภายใน

## 1. When to Use Unsafe

### unsafe blocks ทำอะไรได้บ้าง

```rust
// unsafe ให้สิทธิ์เพิ่มเติม 5 อย่าง:
// 1. Dereference raw pointers
// 2. Call unsafe functions
// 3. Access/modify mutable static variables
// 4. Implement unsafe traits
// 5. Access union fields

fn when_to_use_unsafe() {
    // 1. ต้องการ performance สูงสุด (bypass safety checks)
    // 2. FFI (calling C/C++ code)
    // 3. สร้าง safe abstraction ด้วย unsafe internals
    // 4. Implement low-level data structures
    
    println!("Unsafe is a tool, not a feature to avoid entirely!");
    println!("Key: Unsafe blocks must maintain safety invariants");
}

// ตัวอย่าง: safe wrapper around unsafe code
pub struct SafeBuffer {
    data: Vec<u8>,
    len: usize,
}

impl SafeBuffer {
    pub fn new(capacity: usize) -> Self {
        SafeBuffer {
            data: vec![0u8; capacity],
            len: 0,
        }
    }
    
    // Safe public API
    pub fn write(&mut self, bytes: &[u8]) -> usize {
        let available = self.data.len() - self.len;
        let to_write = bytes.len().min(available);
        
        // unsafe internals ที่มี safety invariants ชัดเจน
        unsafe {
            std::ptr::copy_nonoverlapping(
                bytes.as_ptr(),
                self.data.as_mut_ptr().add(self.len),
                to_write,
            );
        }
        
        self.len += to_write;
        to_write
    }
    
    pub fn as_slice(&self) -> &[u8] {
        &self.data[..self.len]
    }
}
```

## 2. Raw Pointers (*const T, *mut T)

### การใช้งาน raw pointers

```rust
fn raw_pointer_basics() {
    let mut x = 42i32;
    
    // สร้าง raw pointers (ไม่ต้อง unsafe)
    let const_ptr: *const i32 = &x;
    let mut_ptr: *mut i32 = &mut x;
    
    println!("Address: {:p}", const_ptr);
    
    // Dereference ต้องการ unsafe
    unsafe {
        println!("Value via const ptr: {}", *const_ptr);
        *mut_ptr = 100;
        println!("After write via mut ptr: {}", *mut_ptr);
    }
    
    println!("x = {}", x);
}

fn raw_pointer_arithmetic() {
    let arr = [1i32, 2, 3, 4, 5];
    let ptr = arr.as_ptr();
    
    unsafe {
        for i in 0..arr.len() {
            // pointer arithmetic
            let element = *ptr.add(i);
            print!("{} ", element);
        }
        println!();
        
        // offset arithmetic
        let third = ptr.offset(2);
        println!("Third element: {}", *third);
    }
}

// Raw pointer กับ heap memory
fn raw_pointer_heap() {
    use std::alloc::{alloc, dealloc, Layout};
    
    unsafe {
        let layout = Layout::array::<i32>(5).unwrap();
        let ptr = alloc(layout) as *mut i32;
        
        if ptr.is_null() {
            panic!("Allocation failed");
        }
        
        // Initialize
        for i in 0..5i32 {
            *ptr.add(i as usize) = i * i;
        }
        
        // Read back
        for i in 0..5 {
            print!("{} ", *ptr.add(i));
        }
        println!();
        
        // Must free!
        dealloc(ptr as *mut u8, layout);
    }
}

// Null pointer check
fn null_pointer_handling() {
    let ptr: *const i32 = std::ptr::null();
    
    if ptr.is_null() {
        println!("Pointer is null");
    }
    
    // NEVER dereference null pointer!
    // unsafe { *ptr }  // Undefined behavior!
}

fn main() {
    raw_pointer_basics();
    raw_pointer_arithmetic();
    raw_pointer_heap();
    null_pointer_handling();
}
```

## 3. Calling Unsafe Functions

### Unsafe functions และ blocks

```rust
// Declaring an unsafe function
unsafe fn dangerous_function(ptr: *mut u8, len: usize) {
    // Caller must ensure ptr is valid and len is correct
    for i in 0..len {
        *ptr.add(i) = 0;
    }
}

// Safe wrapper
fn zero_bytes(slice: &mut [u8]) {
    unsafe {
        dangerous_function(slice.as_mut_ptr(), slice.len());
    }
}

// std unsafe functions
fn std_unsafe_examples() {
    // slice::from_raw_parts - very unsafe
    let arr = [1u32, 2, 3, 4, 5];
    let ptr = arr.as_ptr();
    let len = arr.len();
    
    unsafe {
        let slice = std::slice::from_raw_parts(ptr, len);
        println!("Slice from raw: {:?}", slice);
        
        // String from raw
        let s = "Hello";
        let bytes = s.as_bytes();
        let s2 = std::str::from_utf8_unchecked(bytes);
        println!("String from raw: {}", s2);
    }
}

// transmute - ระวังมากๆ
fn transmute_demo() {
    // transmute ทำการ reinterpret bits
    let f: f32 = 1.0;
    let bits: u32 = unsafe { std::mem::transmute(f) };
    println!("f32 1.0 as u32 bits: {:#010x}", bits);
    
    let restored: f32 = unsafe { std::mem::transmute(bits) };
    println!("Restored: {}", restored);
    
    // safer alternative
    let bits2 = f.to_bits();
    println!("to_bits: {:#010x}", bits2);
}

fn main() {
    let mut data = vec![0u8; 10];
    for i in 0..10 {
        data[i] = i as u8;
    }
    
    zero_bytes(&mut data);
    println!("After zeroing: {:?}", data);
    
    std_unsafe_examples();
    transmute_demo();
}
```

## 4. Implementing Unsafe Traits (Send, Sync)

### Send และ Sync traits

```rust
use std::sync::Arc;
use std::cell::UnsafeCell;

// Raw pointer wrapper ที่ safe ถ้า ใช้งานถูกต้อง
struct ThreadSafePtr<T> {
    ptr: *mut T,
}

// Compiler ไม่ auto-implement Send/Sync สำหรับ raw pointers
// เราต้อง implement เองพร้อมกับ safety guarantees

unsafe impl<T: Send> Send for ThreadSafePtr<T> {}
unsafe impl<T: Sync> Sync for ThreadSafePtr<T> {}

impl<T> ThreadSafePtr<T> {
    fn new(val: T) -> Self {
        ThreadSafePtr {
            ptr: Box::into_raw(Box::new(val)),
        }
    }
    
    unsafe fn get(&self) -> &T {
        &*self.ptr
    }
    
    unsafe fn get_mut(&self) -> &mut T {
        &mut *self.ptr
    }
}

impl<T> Drop for ThreadSafePtr<T> {
    fn drop(&mut self) {
        unsafe {
            drop(Box::from_raw(self.ptr));
        }
    }
}

// Custom mutex implementation (simplified)
struct SimpleMutex<T> {
    data: UnsafeCell<T>,
    locked: std::sync::atomic::AtomicBool,
}

unsafe impl<T: Send> Send for SimpleMutex<T> {}
unsafe impl<T: Send> Sync for SimpleMutex<T> {}

impl<T> SimpleMutex<T> {
    fn new(data: T) -> Self {
        SimpleMutex {
            data: UnsafeCell::new(data),
            locked: std::sync::atomic::AtomicBool::new(false),
        }
    }
    
    fn lock(&self) -> SimpleMutexGuard<T> {
        // spin-wait (ไม่ดีสำหรับ production)
        while self.locked.compare_exchange(
            false,
            true,
            std::sync::atomic::Ordering::Acquire,
            std::sync::atomic::Ordering::Relaxed,
        ).is_err() {
            std::hint::spin_loop();
        }
        
        SimpleMutexGuard { mutex: self }
    }
}

struct SimpleMutexGuard<'a, T> {
    mutex: &'a SimpleMutex<T>,
}

impl<'a, T> std::ops::Deref for SimpleMutexGuard<'a, T> {
    type Target = T;
    
    fn deref(&self) -> &T {
        unsafe { &*self.mutex.data.get() }
    }
}

impl<'a, T> std::ops::DerefMut for SimpleMutexGuard<'a, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.mutex.data.get() }
    }
}

impl<'a, T> Drop for SimpleMutexGuard<'a, T> {
    fn drop(&mut self) {
        self.mutex.locked.store(false, std::sync::atomic::Ordering::Release);
    }
}

fn main() {
    let mutex = Arc::new(SimpleMutex::new(0i32));
    let mut handles = vec![];
    
    for i in 0..10 {
        let m = mutex.clone();
        handles.push(std::thread::spawn(move || {
            let mut guard = m.lock();
            *guard += i;
        }));
    }
    
    for h in handles {
        h.join().unwrap();
    }
    
    println!("Final value: {}", *mutex.lock());
}
```

## 5. FFI (Foreign Function Interface)

### เรียก C functions จาก Rust

```rust
// Cargo.toml:
// [dependencies]
// libc = "0.2"

use std::ffi::{CString, CStr};
use std::os::raw::{c_char, c_int, c_void};

// ประกาศ C functions
extern "C" {
    fn strlen(s: *const c_char) -> usize;
    fn printf(format: *const c_char, ...) -> c_int;
    fn malloc(size: usize) -> *mut c_void;
    fn free(ptr: *mut c_void);
    fn memcpy(dst: *mut c_void, src: *const c_void, n: usize) -> *mut c_void;
    fn abs(x: c_int) -> c_int;
}

fn call_c_strlen() {
    let s = CString::new("Hello, World!").unwrap();
    let len = unsafe { strlen(s.as_ptr()) };
    println!("C strlen: {}", len);
}

fn call_c_abs() {
    let values = [-5, -3, 0, 3, 5];
    for &v in &values {
        let result = unsafe { abs(v) };
        println!("abs({}) = {}", v, result);
    }
}

// Safe wrapper around C malloc/free
struct CMalloc(*mut c_void, usize);

impl CMalloc {
    fn new(size: usize) -> Option<Self> {
        let ptr = unsafe { malloc(size) };
        if ptr.is_null() {
            None
        } else {
            Some(CMalloc(ptr, size))
        }
    }
    
    fn as_ptr(&self) -> *mut c_void {
        self.0
    }
}

impl Drop for CMalloc {
    fn drop(&mut self) {
        unsafe { free(self.0) };
    }
}

// Callback functions
type Callback = extern "C" fn(data: *mut c_void, value: c_int);

extern "C" fn my_callback(data: *mut c_void, value: c_int) {
    let counter = data as *mut i32;
    unsafe {
        *counter += value as i32;
    }
}

fn main() {
    call_c_strlen();
    call_c_abs();
    
    if let Some(mem) = CMalloc::new(1024) {
        println!("Allocated 1024 bytes at {:p}", mem.as_ptr());
        // CMalloc drops here, calling free()
    }
}
```

## 6. Embedding C Code

### Build script สำหรับ C integration

```rust
// build.rs
use std::process::Command;
use std::env;
use std::path::Path;

fn main() {
    // build.rs ตัวอย่างสำหรับ compiling C code
    println!("cargo:rerun-if-changed=src/math.c");
    println!("cargo:rerun-if-changed=src/math.h");
    
    // ใช้ cc crate สำหรับ compile C (ต้อง add cc = "1.0" ใน build-dependencies)
    // cc::Build::new()
    //     .file("src/math.c")
    //     .compile("math");
}

// C code ที่เราจะ call (math.c):
// int add(int a, int b) { return a + b; }
// int multiply(int a, int b) { return a * b; }
// double sqrt_approx(double x) { ... }

// Rust bindings
extern "C" {
    fn add(a: i32, b: i32) -> i32;
    fn multiply(a: i32, b: i32) -> i32;
}

// Safe wrappers
pub fn safe_add(a: i32, b: i32) -> i32 {
    unsafe { add(a, b) }
}

pub fn safe_multiply(a: i32, b: i32) -> i32 {
    unsafe { multiply(a, b) }
}
```

### bindgen สำหรับ auto-generate bindings

```rust
// build.rs สำหรับ bindgen
// ต้อง add bindgen = "0.69" ใน build-dependencies

/*
extern crate bindgen;

use std::path::PathBuf;

fn main() {
    println!("cargo:rerun-if-changed=wrapper.h");
    
    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("Unable to generate bindings");
    
    let out_path = PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
*/

// จากนั้น include bindings ใน main.rs:
// include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

## 7. Practical: Calling C Library

### ตัวอย่างสมบูรณ์: Rust wrapper สำหรับ C math library

```rust
use std::ffi::CString;
use std::os::raw::{c_char, c_double, c_int};

// สมมติว่ามี C library ชื่อ "mathlib"
// จริงๆ เราใช้ functions จาก libc/libm

extern "C" {
    // math functions
    fn sqrt(x: c_double) -> c_double;
    fn pow(base: c_double, exp: c_double) -> c_double;
    fn sin(x: c_double) -> c_double;
    fn cos(x: c_double) -> c_double;
    fn exp(x: c_double) -> c_double;
    fn log(x: c_double) -> c_double;
    fn fabs(x: c_double) -> c_double;
}

// Safe Rust wrapper
pub struct MathLib;

impl MathLib {
    pub fn sqrt(x: f64) -> f64 {
        assert!(x >= 0.0, "sqrt requires non-negative input");
        unsafe { sqrt(x) }
    }
    
    pub fn pow(base: f64, exp: f64) -> f64 {
        unsafe { pow(base, exp) }
    }
    
    pub fn sin(x: f64) -> f64 {
        unsafe { sin(x) }
    }
    
    pub fn cos(x: f64) -> f64 {
        unsafe { cos(x) }
    }
    
    pub fn exp(x: f64) -> f64 {
        unsafe { exp(x) }
    }
    
    pub fn log(x: f64) -> f64 {
        assert!(x > 0.0, "log requires positive input");
        unsafe { log(x) }
    }
    
    pub fn abs(x: f64) -> f64 {
        unsafe { fabs(x) }
    }
}

// Matrix operations using C-style arrays
struct Matrix {
    data: Vec<f64>,
    rows: usize,
    cols: usize,
}

impl Matrix {
    fn new(rows: usize, cols: usize) -> Self {
        Matrix {
            data: vec![0.0; rows * cols],
            rows,
            cols,
        }
    }
    
    fn get(&self, row: usize, col: usize) -> f64 {
        self.data[row * self.cols + col]
    }
    
    fn set(&mut self, row: usize, col: usize, val: f64) {
        self.data[row * self.cols + col] = val;
    }
    
    fn multiply(&self, other: &Matrix) -> Option<Matrix> {
        if self.cols != other.rows {
            return None;
        }
        
        let mut result = Matrix::new(self.rows, other.cols);
        
        for i in 0..self.rows {
            for j in 0..other.cols {
                let mut sum = 0.0;
                for k in 0..self.cols {
                    sum += self.get(i, k) * other.get(k, j);
                }
                result.set(i, j, sum);
            }
        }
        
        Some(result)
    }
    
    fn print(&self) {
        for i in 0..self.rows {
            for j in 0..self.cols {
                print!("{:.2} ", self.get(i, j));
            }
            println!();
        }
    }
}

// Safety documentation
/// SAFETY INVARIANTS:
/// This function assumes:
/// 1. `ptr` points to valid, initialized memory
/// 2. The memory at `ptr` has at least `len` elements
/// 3. The memory is not aliased by any other mutable reference
/// 4. The memory lives at least as long as the returned slice
unsafe fn slice_from_ptr<'a, T>(ptr: *const T, len: usize) -> &'a [T] {
    std::slice::from_raw_parts(ptr, len)
}

fn main() {
    // Test MathLib
    println!("sqrt(2) = {:.6}", MathLib::sqrt(2.0));
    println!("pow(2, 10) = {}", MathLib::pow(2.0, 10.0));
    println!("sin(π/2) = {:.6}", MathLib::sin(std::f64::consts::PI / 2.0));
    println!("cos(0) = {}", MathLib::cos(0.0));
    println!("exp(1) = {:.6}", MathLib::exp(1.0));
    println!("log(e) = {:.6}", MathLib::log(std::f64::consts::E));
    
    // Matrix operations
    let mut a = Matrix::new(2, 3);
    a.set(0, 0, 1.0); a.set(0, 1, 2.0); a.set(0, 2, 3.0);
    a.set(1, 0, 4.0); a.set(1, 1, 5.0); a.set(1, 2, 6.0);
    
    let mut b = Matrix::new(3, 2);
    b.set(0, 0, 7.0); b.set(0, 1, 8.0);
    b.set(1, 0, 9.0); b.set(1, 1, 10.0);
    b.set(2, 0, 11.0); b.set(2, 1, 12.0);
    
    if let Some(result) = a.multiply(&b) {
        println!("\nMatrix multiplication result:");
        result.print();
    }
}
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **When to use unsafe**: เมื่อ need to interface with C, low-level ops, or implement safe abstractions
- **Raw Pointers**: *const T และ *mut T, pointer arithmetic
- **Unsafe Functions**: วิธี declare และ call unsafe functions
- **Send/Sync**: implement unsafe traits สำหรับ custom types
- **FFI**: เรียก C functions จาก Rust
- **Safety Invariants**: การ document ข้อกำหนดของ unsafe code
- **Safe Wrappers**: pattern ในการ wrap unsafe code ด้วย safe API

---

[← Part 092](../part_092/README.md) | [Part 094 →](../part_094/README.md)
