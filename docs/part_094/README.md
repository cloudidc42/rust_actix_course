# Part 094: WebAssembly with Rust

## บทนำ (Introduction)

WebAssembly (WASM) ช่วยให้เราสามารถรัน Rust code ในเบราว์เซอร์ได้ ด้วย performance เกือบเท่า native code ในบทนี้เราจะเรียนรู้วิธี compile Rust เป็น WASM และ integrate กับ JavaScript

## 1. wasm-pack Setup

### การติดตั้งและตั้งค่า

```bash
# ติดตั้ง wasm-pack
curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh

# หรือใช้ cargo
cargo install wasm-pack

# สร้าง project ใหม่
wasm-pack new my-wasm-project
cd my-wasm-project

# build สำหรับ web
wasm-pack build --target web

# build สำหรับ nodejs
wasm-pack build --target nodejs

# build สำหรับ bundler (webpack, etc.)
wasm-pack build --target bundler
```

### Cargo.toml สำหรับ WASM

```toml
[package]
name = "my-wasm-lib"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
js-sys = "0.3"
web-sys = { version = "0.3", features = [
    "Window",
    "Document",
    "Element",
    "HtmlElement",
    "Node",
    "console",
    "Performance",
    "CanvasRenderingContext2d",
    "HtmlCanvasElement",
] }
serde = { version = "1", features = ["derive"] }
serde-wasm-bindgen = "0.6"
getrandom = { version = "0.2", features = ["js"] }

[dev-dependencies]
wasm-bindgen-test = "0.3"

[profile.release]
opt-level = "s"  # optimize for size
lto = true
```

## 2. #[wasm_bindgen] macro

### พื้นฐานของ wasm_bindgen

```rust
use wasm_bindgen::prelude::*;

// Export function to JavaScript
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u64 {
    match n {
        0 => 0,
        1 => 1,
        _ => {
            let mut a = 0u64;
            let mut b = 1u64;
            for _ in 2..=n {
                let next = a + b;
                a = b;
                b = next;
            }
            b
        }
    }
}

// Export with String return
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}! Welcome to Rust WASM!", name)
}

// Export class to JavaScript
#[wasm_bindgen]
pub struct Counter {
    value: i32,
    step: i32,
}

#[wasm_bindgen]
impl Counter {
    #[wasm_bindgen(constructor)]
    pub fn new(initial_value: i32, step: i32) -> Counter {
        Counter {
            value: initial_value,
            step,
        }
    }
    
    pub fn increment(&mut self) {
        self.value += self.step;
    }
    
    pub fn decrement(&mut self) {
        self.value -= self.step;
    }
    
    pub fn reset(&mut self) {
        self.value = 0;
    }
    
    #[wasm_bindgen(getter)]
    pub fn value(&self) -> i32 {
        self.value
    }
    
    pub fn to_string(&self) -> String {
        format!("Counter: {} (step: {})", self.value, self.step)
    }
}

// Logging to browser console
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);
    
    #[wasm_bindgen(js_namespace = console, js_name = log)]
    fn log_u32(a: u32);
}

macro_rules! console_log {
    ($($t:tt)*) => (log(&format_args!($($t)*).to_string()))
}

#[wasm_bindgen]
pub fn run_demo() {
    console_log!("Hello from Rust WASM!");
    console_log!("2 + 3 = {}", add(2, 3));
    console_log!("fib(10) = {}", fibonacci(10));
}
```

## 3. JavaScript Interop

### ใช้งาน JavaScript APIs จาก Rust

```rust
use wasm_bindgen::prelude::*;
use web_sys::{Window, Document, HtmlElement, Element};

// Access window object
#[wasm_bindgen]
pub fn get_window_width() -> f64 {
    let window = web_sys::window().expect("no global window");
    window.inner_width()
        .expect("no window width")
        .as_f64()
        .unwrap_or(0.0)
}

// Manipulate DOM
#[wasm_bindgen]
pub fn create_button(text: &str, color: &str) -> Result<(), JsValue> {
    let window = web_sys::window().unwrap();
    let document = window.document().unwrap();
    let body = document.body().unwrap();
    
    let button = document.create_element("button")?;
    button.set_inner_html(text);
    
    let html_button = button.dyn_into::<web_sys::HtmlButtonElement>()?;
    html_button.style().set_property("background-color", color)?;
    html_button.style().set_property("color", "white")?;
    html_button.style().set_property("padding", "10px 20px")?;
    html_button.style().set_property("border", "none")?;
    html_button.style().set_property("cursor", "pointer")?;
    
    body.append_child(&html_button)?;
    
    Ok(())
}

// Event listeners
use wasm_bindgen::closure::Closure;

#[wasm_bindgen]
pub fn setup_click_handler(element_id: &str) -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let element = document
        .get_element_by_id(element_id)
        .ok_or_else(|| JsValue::from_str("Element not found"))?;
    
    let click_handler = Closure::wrap(Box::new(move || {
        web_sys::console::log_1(&JsValue::from_str("Button clicked!"));
    }) as Box<dyn Fn()>);
    
    element
        .dyn_ref::<web_sys::HtmlElement>()
        .ok_or_else(|| JsValue::from_str("Not an HTML element"))?
        .set_onclick(Some(click_handler.as_ref().unchecked_ref()));
    
    // Forget the closure to prevent it from being dropped
    click_handler.forget();
    
    Ok(())
}

// Working with JavaScript arrays
use js_sys::{Array, Uint8Array};

#[wasm_bindgen]
pub fn process_array(arr: &Array) -> Array {
    let result = Array::new();
    
    for i in 0..arr.length() {
        let val = arr.get(i);
        if let Some(n) = val.as_f64() {
            result.push(&JsValue::from_f64(n * 2.0));
        }
    }
    
    result
}

// Canvas operations
#[wasm_bindgen]
pub fn draw_mandelbrot(canvas_id: &str, width: u32, height: u32) -> Result<(), JsValue> {
    let document = web_sys::window().unwrap().document().unwrap();
    let canvas = document
        .get_element_by_id(canvas_id)
        .unwrap()
        .dyn_into::<web_sys::HtmlCanvasElement>()?;
    
    canvas.set_width(width);
    canvas.set_height(height);
    
    let ctx = canvas
        .get_context("2d")?
        .unwrap()
        .dyn_into::<web_sys::CanvasRenderingContext2d>()?;
    
    let mut pixels = vec![0u8; (width * height * 4) as usize];
    
    for y in 0..height {
        for x in 0..width {
            let cx = (x as f64 / width as f64) * 3.5 - 2.5;
            let cy = (y as f64 / height as f64) * 2.0 - 1.0;
            
            let iterations = mandelbrot_iter(cx, cy, 255);
            let color = iterations * 255 / 255;
            
            let i = ((y * width + x) * 4) as usize;
            pixels[i] = color as u8;     // R
            pixels[i + 1] = 0;           // G
            pixels[i + 2] = color as u8; // B
            pixels[i + 3] = 255;         // A
        }
    }
    
    let data = web_sys::ImageData::new_with_u8_clamped_array_and_sh(
        wasm_bindgen::Clamped(&pixels),
        width,
        height,
    )?;
    
    ctx.put_image_data(&data, 0.0, 0.0)?;
    
    Ok(())
}

fn mandelbrot_iter(cx: f64, cy: f64, max_iter: u32) -> u32 {
    let mut x = 0.0f64;
    let mut y = 0.0f64;
    let mut iter = 0;
    
    while x * x + y * y <= 4.0 && iter < max_iter {
        let xtemp = x * x - y * y + cx;
        y = 2.0 * x * y + cy;
        x = xtemp;
        iter += 1;
    }
    
    iter
}
```

## 4. wasm-bindgen-futures

### Async/Await ใน WASM

```rust
use wasm_bindgen::prelude::*;
use wasm_bindgen_futures::JsFuture;
use web_sys::{Request, RequestInit, RequestMode, Response};
use js_sys::Promise;

// Async function ที่ expose ไปยัง JavaScript
#[wasm_bindgen]
pub async fn fetch_data(url: String) -> Result<JsValue, JsValue> {
    let mut opts = RequestInit::new();
    opts.method("GET");
    opts.mode(RequestMode::Cors);
    
    let request = Request::new_with_str_and_init(&url, &opts)?;
    
    let window = web_sys::window().unwrap();
    let response_value = JsFuture::from(window.fetch_with_request(&request)).await?;
    
    let response: Response = response_value.dyn_into()?;
    let json = JsFuture::from(response.json()?).await?;
    
    Ok(json)
}

// Sleeping in WASM
#[wasm_bindgen]
pub async fn delayed_computation(ms: i32) -> String {
    let promise = js_sys::Promise::new(&mut |resolve, _| {
        web_sys::window()
            .unwrap()
            .set_timeout_with_callback_and_timeout_and_arguments_0(&resolve, ms)
            .unwrap();
    });
    
    JsFuture::from(promise).await.unwrap();
    
    format!("Completed after {}ms delay", ms)
}

// Running multiple async operations
#[wasm_bindgen]
pub async fn parallel_fetch(urls: Vec<JsValue>) -> js_sys::Array {
    let futures: Vec<_> = urls.iter()
        .filter_map(|url| url.as_string())
        .map(|url| fetch_data(url))
        .collect();
    
    let results = js_sys::Array::new();
    
    for future in futures {
        match future.await {
            Ok(data) => results.push(&data),
            Err(e) => results.push(&e),
        }
    }
    
    results
}
```

## 5. Sharing Types Between JS and Rust

### Serde serialization

```rust
use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone, Debug)]
pub struct User {
    pub id: u32,
    pub name: String,
    pub email: String,
    pub age: u32,
    pub tags: Vec<String>,
}

#[derive(Serialize, Deserialize, Debug)]
pub struct UserList {
    pub users: Vec<User>,
    pub total: usize,
    pub page: u32,
}

#[wasm_bindgen]
pub fn create_user(id: u32, name: &str, email: &str, age: u32) -> JsValue {
    let user = User {
        id,
        name: name.to_string(),
        email: email.to_string(),
        age,
        tags: vec!["rust".to_string(), "wasm".to_string()],
    };
    
    serde_wasm_bindgen::to_value(&user).unwrap()
}

#[wasm_bindgen]
pub fn process_users(users_js: JsValue) -> JsValue {
    let users: Vec<User> = serde_wasm_bindgen::from_value(users_js).unwrap();
    
    let processed: Vec<User> = users.into_iter()
        .filter(|u| u.age >= 18)
        .map(|mut u| {
            u.name = u.name.to_uppercase();
            u.tags.push("processed".to_string());
            u
        })
        .collect();
    
    let result = UserList {
        total: processed.len(),
        users: processed,
        page: 1,
    };
    
    serde_wasm_bindgen::to_value(&result).unwrap()
}

// Complex data types
#[derive(Serialize, Deserialize)]
pub struct Config {
    pub width: u32,
    pub height: u32,
    pub scale: f64,
    pub colors: Vec<[u8; 4]>,
    pub properties: std::collections::HashMap<String, String>,
}

#[wasm_bindgen]
pub fn validate_config(config_js: JsValue) -> Result<bool, JsValue> {
    let config: Config = serde_wasm_bindgen::from_value(config_js)
        .map_err(|e| JsValue::from_str(&e.to_string()))?;
    
    let valid = config.width > 0 && config.height > 0 && config.scale > 0.0;
    Ok(valid)
}
```

## 6. Performance Benefits

### WASM vs JavaScript performance

```rust
use wasm_bindgen::prelude::*;

// Heavy computation - เร็วกว่า JS มาก
#[wasm_bindgen]
pub fn find_primes(limit: u32) -> Vec<u32> {
    let mut sieve = vec![true; (limit + 1) as usize];
    sieve[0] = false;
    if limit >= 1 {
        sieve[1] = false;
    }
    
    let mut i = 2;
    while i * i <= limit {
        if sieve[i as usize] {
            let mut j = i * i;
            while j <= limit {
                sieve[j as usize] = false;
                j += i;
            }
        }
        i += 1;
    }
    
    sieve.iter().enumerate()
        .filter_map(|(i, &is_prime)| {
            if is_prime { Some(i as u32) } else { None }
        })
        .collect()
}

// Image processing (fast with WASM)
#[wasm_bindgen]
pub fn apply_grayscale(pixels: &mut [u8]) {
    for i in (0..pixels.len()).step_by(4) {
        let r = pixels[i] as f32;
        let g = pixels[i + 1] as f32;
        let b = pixels[i + 2] as f32;
        
        let gray = (0.299 * r + 0.587 * g + 0.114 * b) as u8;
        
        pixels[i] = gray;
        pixels[i + 1] = gray;
        pixels[i + 2] = gray;
        // pixels[i + 3] unchanged (alpha)
    }
}

// Cryptography
#[wasm_bindgen]
pub fn sha256_hash(data: &[u8]) -> Vec<u8> {
    // Simplified - ใน production ใช้ sha2 crate
    let mut hash = [0u8; 32];
    for (i, &byte) in data.iter().enumerate() {
        hash[i % 32] ^= byte;
        hash[i % 32] = hash[i % 32].wrapping_add(i as u8);
    }
    hash.to_vec()
}

// Performance benchmark
#[wasm_bindgen]
pub fn benchmark_matrix_multiply(size: usize) -> f64 {
    let a: Vec<f64> = (0..size * size).map(|i| i as f64).collect();
    let b: Vec<f64> = (0..size * size).map(|i| i as f64 * 0.5).collect();
    let mut c: Vec<f64> = vec![0.0; size * size];
    
    for i in 0..size {
        for j in 0..size {
            let mut sum = 0.0;
            for k in 0..size {
                sum += a[i * size + k] * b[k * size + j];
            }
            c[i * size + j] = sum;
        }
    }
    
    c[0] // return first element as proof of work
}
```

## 7. Practical: Rust Functions in Browser

### สร้าง full WASM application

```rust
use wasm_bindgen::prelude::*;
use std::collections::HashMap;

// Simple key-value store in WASM
#[wasm_bindgen]
pub struct Store {
    data: HashMap<String, String>,
}

#[wasm_bindgen]
impl Store {
    #[wasm_bindgen(constructor)]
    pub fn new() -> Store {
        Store {
            data: HashMap::new(),
        }
    }
    
    pub fn set(&mut self, key: &str, value: &str) {
        self.data.insert(key.to_string(), value.to_string());
    }
    
    pub fn get(&self, key: &str) -> Option<String> {
        self.data.get(key).cloned()
    }
    
    pub fn delete(&mut self, key: &str) -> bool {
        self.data.remove(key).is_some()
    }
    
    pub fn keys(&self) -> Vec<JsValue> {
        self.data.keys()
            .map(|k| JsValue::from_str(k))
            .collect()
    }
    
    pub fn len(&self) -> usize {
        self.data.len()
    }
    
    pub fn is_empty(&self) -> bool {
        self.data.is_empty()
    }
    
    pub fn clear(&mut self) {
        self.data.clear();
    }
    
    pub fn to_json(&self) -> String {
        let pairs: Vec<String> = self.data.iter()
            .map(|(k, v)| format!("\"{}\":\"{}\"", k, v))
            .collect();
        format!("{{{}}}", pairs.join(","))
    }
}

// Game of Life
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<u8>,
}

#[wasm_bindgen]
impl Universe {
    #[wasm_bindgen(constructor)]
    pub fn new(width: u32, height: u32) -> Universe {
        let cells: Vec<u8> = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 { 1 } else { 0 }
            })
            .collect();
        
        Universe { width, height, cells }
    }
    
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();
        
        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);
                
                next[idx] = match (cell, live_neighbors) {
                    (1, x) if x < 2 => 0,
                    (1, 2) | (1, 3) => 1,
                    (1, x) if x > 3 => 0,
                    (0, 3) => 1,
                    (other, _) => other,
                };
            }
        }
        
        self.cells = next;
    }
    
    pub fn cells(&self) -> *const u8 {
        self.cells.as_ptr()
    }
    
    pub fn width(&self) -> u32 {
        self.width
    }
    
    pub fn height(&self) -> u32 {
        self.height
    }
    
    fn index(&self, row: u32, col: u32) -> usize {
        (row * self.width + col) as usize
    }
    
    fn live_neighbor_count(&self, row: u32, col: u32) -> u8 {
        let mut count = 0;
        
        for delta_row in [self.height - 1, 0, 1] {
            for delta_col in [self.width - 1, 0, 1] {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }
                
                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (col + delta_col) % self.width;
                let idx = self.index(neighbor_row, neighbor_col);
                count += self.cells[idx];
            }
        }
        
        count
    }
}

// HTML page ที่ใช้งาน WASM
#[wasm_bindgen(start)]
pub fn main() {
    // ตั้งค่า panic hook สำหรับ better error messages
    console_error_panic_hook::set_once();
    
    web_sys::console::log_1(&JsValue::from_str("WASM module loaded!"));
}
```

### HTML ตัวอย่างสำหรับใช้ WASM

```html
<!DOCTYPE html>
<html>
<head>
    <title>Rust WASM Demo</title>
</head>
<body>
    <h1>Rust WebAssembly Demo</h1>
    <canvas id="game-of-life" width="640" height="640"></canvas>
    
    <script type="module">
        import init, { Universe, find_primes, Counter } from './pkg/my_wasm_lib.js';
        
        async function main() {
            await init();
            
            // Test Counter
            const counter = new Counter(0, 5);
            console.log(counter.to_string());
            counter.increment();
            counter.increment();
            console.log(`Value: ${counter.value}`);
            
            // Find primes
            const primes = find_primes(100);
            console.log('Primes up to 100:', primes);
            
            // Game of Life
            const universe = new Universe(64, 64);
            const canvas = document.getElementById('game-of-life');
            const ctx = canvas.getContext('2d');
            
            const CELL_SIZE = 10;
            
            function drawCells() {
                const cellsPtr = universe.cells();
                // Use WebAssembly memory directly
                const cells = new Uint8Array(memory.buffer, cellsPtr, universe.width() * universe.height());
                
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                
                for (let row = 0; row < universe.height(); row++) {
                    for (let col = 0; col < universe.width(); col++) {
                        const idx = row * universe.width() + col;
                        ctx.fillStyle = cells[idx] === 1 ? '#000000' : '#FFFFFF';
                        ctx.fillRect(col * CELL_SIZE, row * CELL_SIZE, CELL_SIZE, CELL_SIZE);
                    }
                }
            }
            
            function renderLoop() {
                universe.tick();
                drawCells();
                requestAnimationFrame(renderLoop);
            }
            
            requestAnimationFrame(renderLoop);
        }
        
        main();
    </script>
</body>
</html>
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **wasm-pack**: เครื่องมือสำหรับ build Rust เป็น WebAssembly
- **#[wasm_bindgen]**: macro สำหรับ expose Rust types และ functions ไปยัง JavaScript
- **JavaScript Interop**: การใช้งาน DOM, events, Web APIs
- **Async WASM**: ใช้ wasm-bindgen-futures สำหรับ async operations
- **Type Sharing**: Serde serialization สำหรับ complex types
- **Performance**: WASM เร็วกว่า JavaScript สำหรับ heavy computations
- **Game of Life**: ตัวอย่าง practical application

---

[← Part 093](../part_093/README.md) | [Part 095 →](../part_095/README.md)
