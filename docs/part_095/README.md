# Part 095: Embedded Systems with Rust

## บทนำ (Introduction)

Rust เหมาะอย่างยิ่งสำหรับ embedded systems เพราะมี zero-cost abstractions, ไม่มี garbage collector, และ memory safety โดยไม่ต้องใช้ runtime ขนาดใหญ่ ในบทนี้เราจะเรียนรู้การพัฒนาสำหรับ microcontrollers

## 1. no_std Environment

### โปรแกรม Rust โดยไม่มี standard library

```rust
// lib.rs หรือ main.rs
#![no_std]
#![no_main]

// ไม่มี std library - ต้องประกาศ panic handler เอง
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    // ใน embedded - loop ตลอดไป
    loop {}
}

// เข้าถึง core library (subset ของ std)
use core::fmt::Write;
use core::mem;
use core::ptr;
use core::sync::atomic::{AtomicBool, Ordering};

// ตัวอย่าง data structures ที่ใช้ได้ใน no_std
fn core_types_demo() {
    // Vec ไม่มี - ต้องใช้ fixed-size arrays
    let arr: [u8; 64] = [0; 64];
    
    // String ไม่มี - ต้องใช้ &str หรือ heapless::String
    let s: &str = "Hello, embedded!";
    
    // HashMap ไม่มี - ต้องใช้ heapless::FnvIndexMap หรือ linear search
    let _ = s.len();
}
```

### Cargo.toml สำหรับ STM32

```toml
[package]
name = "stm32-blink"
version = "0.1.0"
edition = "2021"

[dependencies]
# HAL สำหรับ STM32F4
stm32f4xx-hal = { version = "0.14", features = ["stm32f411", "rt"] }
# Cortex-M support
cortex-m = "0.7"
cortex-m-rt = "0.7"
# นับ panic handler
panic-halt = "0.2"
# ไม่ใช้ allocator
heapless = "0.7"

[profile.release]
opt-level = "z"
debug = true
lto = true
codegen-units = 1

# Target สำหรับ ARM Cortex-M4F
[build]
target = "thumbv7em-none-eabihf"
```

## 2. Embedded HAL Traits

### Hardware Abstraction Layer

```rust
#![no_std]
#![no_main]

use embedded_hal::digital::v2::{OutputPin, InputPin, ToggleableOutputPin};
use embedded_hal::blocking::delay::DelayMs;

// Generic function ที่ทำงานกับ hardware ใดก็ได้
fn blink<Pin, Delay>(led: &mut Pin, delay: &mut Delay, count: u32)
where
    Pin: OutputPin + ToggleableOutputPin,
    Delay: DelayMs<u32>,
{
    for _ in 0..count {
        led.toggle().ok();
        delay.delay_ms(500u32);
        led.toggle().ok();
        delay.delay_ms(500u32);
    }
}

// Generic I2C reader
fn read_sensor<I2C, Error>(
    i2c: &mut I2C,
    address: u8,
    register: u8,
) -> Result<u8, Error>
where
    I2C: embedded_hal::blocking::i2c::WriteRead<Error = Error>,
{
    let mut buffer = [0u8; 1];
    i2c.write_read(address, &[register], &mut buffer)?;
    Ok(buffer[0])
}

// Generic SPI writer
fn write_display<SPI, CS, Error>(
    spi: &mut SPI,
    cs: &mut CS,
    data: &[u8],
) -> Result<(), Error>
where
    SPI: embedded_hal::blocking::spi::Write<u8, Error = Error>,
    CS: OutputPin,
{
    cs.set_low().ok();
    spi.write(data)?;
    cs.set_high().ok();
    Ok(())
}
```

## 3. Interrupt Handling

### ตั้งค่า interrupts

```rust
#![no_std]
#![no_main]

use cortex_m::interrupt::Mutex;
use cortex_m_rt::entry;
use core::cell::RefCell;
use core::sync::atomic::{AtomicBool, AtomicU32, Ordering};

// Global state สำหรับ interrupt handlers
static BUTTON_PRESSED: AtomicBool = AtomicBool::new(false);
static TICK_COUNT: AtomicU32 = AtomicU32::new(0);

// GPIO interrupt handler
#[cortex_m_rt::interrupt]
fn EXTI0() {
    BUTTON_PRESSED.store(true, Ordering::Release);
    
    // Clear interrupt flag (hardware-specific)
    // EXTI.pr.write(|w| w.pr0().set_bit());
}

// Timer interrupt handler
#[cortex_m_rt::interrupt]
fn TIM2() {
    TICK_COUNT.fetch_add(1, Ordering::Relaxed);
    
    // Clear interrupt flag
    // tim2.sr.modify(|_, w| w.uif().clear_bit());
}

// Safe sharing ระหว่าง interrupt และ main code
use cortex_m::interrupt;

static SHARED_DATA: Mutex<RefCell<Option<u32>>> = Mutex::new(RefCell::new(None));

fn update_shared(value: u32) {
    interrupt::free(|cs| {
        *SHARED_DATA.borrow(cs).borrow_mut() = Some(value);
    });
}

fn read_shared() -> Option<u32> {
    interrupt::free(|cs| {
        *SHARED_DATA.borrow(cs).borrow()
    })
}

// RTIC framework (Real-Time Interrupt-driven Concurrency)
// #[rtic::app(device = stm32f4xx_hal::pac, peripherals = true)]
// mod app {
//     #[resources]
//     struct Resources {
//         led: Led,
//         button: Button,
//         #[init(0)]
//         count: u32,
//     }
//
//     #[task(resources = [led, count], schedule = [blink])]
//     fn blink(cx: blink::Context) {
//         cx.resources.led.toggle().unwrap();
//         *cx.resources.count += 1;
//         cx.schedule.blink(cx.scheduled + PERIOD.cycles()).unwrap();
//     }
//
//     extern "C" {
//         fn TIM2();
//     }
// }
```

## 4. Memory-mapped I/O

### Direct hardware register access

```rust
#![no_std]

use core::ptr::{read_volatile, write_volatile};

// Base addresses สำหรับ STM32F4
const RCC_BASE: u32 = 0x4002_3800;
const GPIOA_BASE: u32 = 0x4002_0000;
const GPIOC_BASE: u32 = 0x4002_0800;

// Register offsets
const RCC_AHB1ENR: u32 = 0x30;
const GPIO_MODER: u32 = 0x00;
const GPIO_ODR: u32 = 0x14;
const GPIO_IDR: u32 = 0x10;
const GPIO_BSRR: u32 = 0x18;

// Helper functions สำหรับ memory-mapped I/O
unsafe fn read_reg(base: u32, offset: u32) -> u32 {
    read_volatile((base + offset) as *const u32)
}

unsafe fn write_reg(base: u32, offset: u32, value: u32) {
    write_volatile((base + offset) as *mut u32, value);
}

unsafe fn modify_reg<F>(base: u32, offset: u32, f: F)
where
    F: FnOnce(u32) -> u32,
{
    let current = read_reg(base, offset);
    write_reg(base, offset, f(current));
}

// เปิด clock สำหรับ GPIOA
unsafe fn enable_gpioa_clock() {
    modify_reg(RCC_BASE, RCC_AHB1ENR, |r| r | (1 << 0));
}

// ตั้งค่า PA5 เป็น output (LED บน Nucleo board)
unsafe fn configure_led() {
    modify_reg(GPIOA_BASE, GPIO_MODER, |r| {
        // MODER5 = 01 (output mode)
        let mask = 0b11 << 10; // bits 11:10
        (r & !mask) | (0b01 << 10)
    });
}

// เปิด LED
unsafe fn led_on() {
    write_reg(GPIOA_BASE, GPIO_BSRR, 1 << 5); // Set bit 5
}

// ปิด LED
unsafe fn led_off() {
    write_reg(GPIOA_BASE, GPIO_BSRR, 1 << (5 + 16)); // Reset bit 5
}

// Type-safe register abstraction
struct GpioPort {
    base: u32,
}

impl GpioPort {
    const fn new(base: u32) -> Self {
        GpioPort { base }
    }
    
    fn set_pin_output(&self, pin: u8) {
        assert!(pin < 16);
        unsafe {
            modify_reg(self.base, GPIO_MODER, |r| {
                let shift = (pin as u32) * 2;
                let mask = 0b11 << shift;
                (r & !mask) | (0b01 << shift)
            });
        }
    }
    
    fn set_pin_high(&self, pin: u8) {
        assert!(pin < 16);
        unsafe {
            write_reg(self.base, GPIO_BSRR, 1 << pin as u32);
        }
    }
    
    fn set_pin_low(&self, pin: u8) {
        assert!(pin < 16);
        unsafe {
            write_reg(self.base, GPIO_BSRR, 1 << (pin as u32 + 16));
        }
    }
    
    fn read_pin(&self, pin: u8) -> bool {
        assert!(pin < 16);
        unsafe {
            let idr = read_reg(self.base, GPIO_IDR);
            (idr & (1 << pin as u32)) != 0
        }
    }
}

static GPIOA: GpioPort = GpioPort::new(GPIOA_BASE);
static GPIOC: GpioPort = GpioPort::new(GPIOC_BASE);
```

## 5. Serial Communication (UART)

### UART communication

```rust
#![no_std]

use core::fmt::Write;
use heapless::String;

// UART registers (simplified)
const USART2_BASE: u32 = 0x4000_4400;
const USART_SR: u32 = 0x00;
const USART_DR: u32 = 0x04;
const USART_BRR: u32 = 0x08;
const USART_CR1: u32 = 0x0C;

// UART driver
struct Uart {
    base: u32,
}

impl Uart {
    unsafe fn init(&self, baud_rate: u32, clock_hz: u32) {
        // Calculate baud rate divisor
        let div = clock_hz / baud_rate;
        
        // Set baud rate
        core::ptr::write_volatile((self.base + USART_BRR) as *mut u32, div);
        
        // Enable UART, TX, RX
        core::ptr::write_volatile(
            (self.base + USART_CR1) as *mut u32,
            (1 << 13) | (1 << 3) | (1 << 2),
        );
    }
    
    unsafe fn send_byte(&self, byte: u8) {
        // Wait for TX empty
        while (core::ptr::read_volatile((self.base + USART_SR) as *const u32) & (1 << 7)) == 0 {}
        
        // Send byte
        core::ptr::write_volatile((self.base + USART_DR) as *mut u32, byte as u32);
    }
    
    unsafe fn receive_byte(&self) -> u8 {
        // Wait for data
        while (core::ptr::read_volatile((self.base + USART_SR) as *const u32) & (1 << 5)) == 0 {}
        
        core::ptr::read_volatile((self.base + USART_DR) as *const u32) as u8
    }
    
    fn send_str(&self, s: &str) {
        for byte in s.bytes() {
            unsafe { self.send_byte(byte) };
        }
    }
}

impl Write for Uart {
    fn write_str(&mut self, s: &str) -> core::fmt::Result {
        self.send_str(s);
        Ok(())
    }
}

// ใช้งาน UART
fn uart_demo() {
    let mut uart = Uart { base: USART2_BASE };
    
    // Write formatted output
    let _ = write!(uart, "Hello, UART!\r\n");
    let _ = write!(uart, "Temperature: {}°C\r\n", 25);
    
    // heapless String (no heap allocation)
    let mut buf: String<64> = String::new();
    write!(buf, "Count: {}", 42).ok();
    uart.send_str(&buf);
}
```

## 6. GPIO Control

### LED Blink และ Button reading

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
    // Get access to device peripherals
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::peripheral::Peripherals::take().unwrap();
    
    // Setup clock
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.sysclk(84.MHz()).freeze();
    
    // Get GPIO pins
    let gpioa = dp.GPIOA.split();
    let gpioc = dp.GPIOC.split();
    
    // LED on PA5 (Nucleo board built-in LED)
    let mut led = gpioa.pa5.into_push_pull_output();
    
    // Button on PC13 (Nucleo board built-in button)
    let button = gpioc.pc13.into_pull_up_input();
    
    // Setup delay
    let mut delay = cp.SYST.delay(&clocks);
    
    let mut led_state = false;
    let mut blink_rate_ms = 500u32;
    
    loop {
        // Read button state
        if button.is_low() {
            // Button pressed - speed up blink
            blink_rate_ms = 100;
        } else {
            blink_rate_ms = 500;
        }
        
        // Toggle LED
        if led_state {
            led.set_high();
        } else {
            led.set_low();
        }
        led_state = !led_state;
        
        delay.delay_ms(blink_rate_ms);
    }
}
```

## 7. Practical: Complete LED Blink on STM32

### โปรแกรม LED blink สมบูรณ์

```rust
//! # STM32 LED Blink Example
//! 
//! This program blinks the LED on PA5 with configurable patterns
//! and responds to button presses on PC13.

#![no_std]
#![no_main]

use cortex_m::asm;
use cortex_m_rt::entry;
use panic_halt as _;
use stm32f4xx_hal::{
    gpio::{Output, PushPull, PA5},
    pac,
    prelude::*,
    timer::Timer,
};
use heapless::Vec;

// LED blink patterns
#[derive(Clone, Copy)]
enum Pattern {
    Slow,
    Fast,
    SOS,
    Heartbeat,
}

struct LedController {
    patterns: [(&'static [u32], u32); 4], // (on_ms, off_ms) sequences
    current_pattern: usize,
}

impl LedController {
    fn new() -> Self {
        LedController {
            patterns: [
                (&[500, 500], 0),        // Slow
                (&[100, 100], 0),        // Fast
                (&[100, 100, 100, 100, 100, 700, 300, 700, 300, 700, 700, 100, 100, 100, 100, 100, 1000], 0), // SOS
                (&[100, 100, 300, 700], 0), // Heartbeat
            ],
            current_pattern: 0,
        }
    }
    
    fn next_pattern(&mut self) {
        self.current_pattern = (self.current_pattern + 1) % 4;
        self.patterns[self.current_pattern].1 = 0; // reset index
    }
    
    fn get_delay(&mut self) -> u32 {
        let (pattern, idx) = &mut self.patterns[self.current_pattern];
        let delay = pattern[*idx as usize % pattern.len()];
        *idx += 1;
        delay
    }
}

// เพิ่ม simple debounce
struct Button {
    last_state: bool,
    debounce_count: u32,
    pressed: bool,
}

impl Button {
    fn new() -> Self {
        Button {
            last_state: false,
            debounce_count: 0,
            pressed: false,
        }
    }
    
    fn update(&mut self, current_state: bool) -> bool {
        if current_state != self.last_state {
            self.debounce_count += 1;
            if self.debounce_count >= 10 {
                self.last_state = current_state;
                self.debounce_count = 0;
                if current_state {
                    self.pressed = true;
                    return true;
                }
            }
        } else {
            self.debounce_count = 0;
        }
        false
    }
    
    fn take_pressed(&mut self) -> bool {
        let p = self.pressed;
        self.pressed = false;
        p
    }
}

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    let cp = cortex_m::peripheral::Peripherals::take().unwrap();
    
    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.sysclk(84.MHz()).freeze();
    
    let gpioa = dp.GPIOA.split();
    let gpioc = dp.GPIOC.split();
    
    let mut led = gpioa.pa5.into_push_pull_output();
    let button_pin = gpioc.pc13.into_pull_up_input();
    
    let mut delay = cp.SYST.delay(&clocks);
    
    let mut controller = LedController::new();
    let mut button = Button::new();
    let mut led_on = false;
    
    loop {
        // Read button (active low)
        let btn_state = button_pin.is_low();
        if button.update(btn_state) {
            if button.take_pressed() {
                controller.next_pattern();
            }
        }
        
        // Toggle LED
        if led_on {
            led.set_high();
        } else {
            led.set_low();
        }
        led_on = !led_on;
        
        // Get delay for current pattern
        let ms = controller.get_delay();
        delay.delay_ms(ms);
    }
}

// Memory layout file (memory.x):
/*
MEMORY
{
    FLASH : ORIGIN = 0x08000000, LENGTH = 512K
    RAM : ORIGIN = 0x20000000, LENGTH = 128K
}
*/
```

## 8. RTOS Integration

### FreeRTOS-like scheduling

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _;

// ใช้ Embassy - async/await สำหรับ embedded
// Cargo.toml:
// embassy-executor = { version = "*", features = ["arch-cortex-m"] }
// embassy-time = { version = "*" }
// embassy-stm32 = { version = "*", features = ["stm32f411ce"] }

// ตัวอย่าง task management (conceptual)
struct TaskControl {
    tasks: heapless::Vec<Task, 8>,
    current_task: usize,
    tick: u32,
}

struct Task {
    id: u8,
    period_ms: u32,
    last_run: u32,
    func: fn(),
}

impl TaskControl {
    fn new() -> Self {
        TaskControl {
            tasks: heapless::Vec::new(),
            current_task: 0,
            tick: 0,
        }
    }
    
    fn add_task(&mut self, id: u8, period_ms: u32, func: fn()) -> bool {
        self.tasks.push(Task { id, period_ms, last_run: 0, func }).is_ok()
    }
    
    fn tick(&mut self) {
        self.tick += 1;
        
        for task in self.tasks.iter_mut() {
            if self.tick - task.last_run >= task.period_ms {
                task.last_run = self.tick;
                (task.func)();
            }
        }
    }
}

fn sensor_task() {
    // อ่านค่าจาก sensor
}

fn display_task() {
    // อัปเดต display
}

fn led_task() {
    // toggle LED
}

// Embassy async example (ถ้าใช้ Embassy framework)
/*
use embassy_executor::Spawner;
use embassy_stm32::gpio::{Level, Output, Speed};
use embassy_time::{Duration, Timer};

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let p = embassy_stm32::init(Default::default());
    
    spawner.spawn(blink_task(p.PA5)).unwrap();
    spawner.spawn(button_task(p.PC13)).unwrap();
}

#[embassy_executor::task]
async fn blink_task(pin: embassy_stm32::gpio::AnyPin) {
    let mut led = Output::new(pin, Level::Low, Speed::Low);
    
    loop {
        led.set_high();
        Timer::after(Duration::from_millis(500)).await;
        led.set_low();
        Timer::after(Duration::from_millis(500)).await;
    }
}

#[embassy_executor::task]
async fn button_task(pin: embassy_stm32::gpio::AnyPin) {
    use embassy_stm32::gpio::{Input, Pull};
    let button = Input::new(pin, Pull::Up);
    
    loop {
        button.wait_for_falling_edge().await;
        defmt::info!("Button pressed!");
    }
}
*/
```

## สรุป (Summary)

ในบทนี้เราได้เรียนรู้:
- **no_std**: การเขียน Rust โดยไม่มี standard library
- **Embedded HAL**: traits ที่ทำให้ code ทำงานได้กับ hardware หลายชนิด
- **Interrupts**: การจัดการ hardware interrupts อย่างปลอดภัย
- **Memory-mapped I/O**: การเข้าถึง hardware registers โดยตรง
- **UART**: Serial communication
- **GPIO**: การควบคุม digital I/O
- **RTOS**: การ integrate กับ real-time operating systems
- **LED Blink**: ตัวอย่าง complete application สำหรับ STM32

---

[← Part 094](../part_094/README.md) | [Part 096 →](../part_096/README.md)
