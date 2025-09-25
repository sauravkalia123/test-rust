# Rust Master Revision Cheat-Sheet (Interview Focused)

Compact guide with **concepts + code examples**.  
Use this for fast revision before interviews.

---

## 1. Ownership & Borrowing

Rust memory safety ka base concept hai.

### Key Points
- **Ownership** → ek variable ek value ka single owner hota hai.
- **Move** → ownership transfer hoti hai jab assign karte ho.
- **Borrowing** → reference dena without ownership transfer.
- `&T` immutable borrow (multiple allowed).
- `&mut T` mutable borrow (exclusive).
- **Lifetimes** → compiler ensures reference valid hai.

### Example
```rust
fn ownership_demo() {
    let s = String::from("hello"); // owner: s
    let r = &s;                    // borrow
    println!("{}", r);

    let s2 = s; // move ownership to s2
    println!("{}", s2);

    // println!("{}", s); // ❌ error, s moved
}
```

---

## 2. Types & Collections

### Key Points
- **Array**: `[T; N]` fixed-size.
- **Slice**: `&[T]` view into array/vec.
- **Vec<T>**: growable heap array.
- **String**: `String` (owned), `&str` (borrowed).
- **HashMap<K,V>** in `std::collections`.

### Example
```rust
use std::collections::HashMap;

fn types_demo() {
    let arr: [i32; 3] = [1, 2, 3];
    let slice: &[i32] = &arr[0..2];
    let mut v: Vec<i32> = vec![1, 2, 3];
    v.push(4);

    let mut map = HashMap::new();
    map.insert("apple", 3);
    println!("Slice: {:?}, Vec: {:?}, Map: {:?}", slice, v, map);
}
```

---

## 3. Enums & Pattern Matching

### Key Points
- Enums can carry data.
- `match` is exhaustive.
- `if let` and `while let` simplify pattern matching.

### Example
```rust
enum Value {
    Int(i32),
    Str(String),
    Float(f64),
}

fn enum_demo(x: Value) {
    match x {
        Value::Int(n) => println!("int {}", n),
        Value::Str(s) => println!("str {}", s),
        _ => println!("something else"),
    }
}
```

---

## 4. Structs, impl, Constructors & Drop

### Key Points
- Structs = user-defined types.
- `impl` block for methods.
- `new()` idiomatic constructor.
- Implement `Drop` for destructor logic.

### Example
```rust
pub struct Circle {
    radius: f64,
}

impl Circle {
    pub fn new(r: f64) -> Self {
        Circle { radius: r }
    }
    pub fn area(&self) -> f64 {
        3.14 * self.radius * self.radius
    }
}

impl Drop for Circle {
    fn drop(&mut self) {
        println!("Dropping circle with radius {}", self.radius);
    }
}
```

---

## 5. Traits & Polymorphism

### Key Points
- Traits = interface (like pure virtual).
- Default implementations allowed.
- Dynamic dispatch with `Box<dyn Trait>`.

### Example
```rust
trait Shape {
    fn area(&self) -> f64;
    fn name(&self) -> &str;
}

struct Rectangle { w: f64, h: f64 }
struct Circle { r: f64 }

impl Shape for Rectangle {
    fn area(&self) -> f64 { self.w * self.h }
    fn name(&self) -> &str { "Rectangle" }
}
impl Shape for Circle {
    fn area(&self) -> f64 { 3.14 * self.r * self.r }
    fn name(&self) -> &str { "Circle" }
}

fn trait_demo() {
    let shapes: Vec<Box<dyn Shape>> = vec![
        Box::new(Rectangle { w: 2.0, h: 3.0 }),
        Box::new(Circle { r: 1.0 }),
    ];
    for s in shapes {
        println!("{} area = {}", s.name(), s.area());
    }
}
```

---

## 6. Smart Pointers

### Key Points
- `Box<T>`: heap allocate, single owner.
- `Rc<T>`: reference-counted single-threaded.
- `Arc<T>`: atomic Rc for multi-threaded.
- `RefCell<T>`: runtime borrow check (single-threaded).
- `Mutex<T>`/`RwLock<T>`: thread-safe shared access.

### Example
```rust
use std::rc::Rc;
use std::sync::{Arc, Mutex};

fn smart_demo() {
    let b = Box::new(5);
    println!("Box = {}", b);

    let r = Rc::new(vec![1,2,3]);
    let r2 = Rc::clone(&r);
    println!("Rc count = {}", Rc::strong_count(&r));

    let a = Arc::new(Mutex::new(0));
    {
        let mut g = a.lock().unwrap();
        *g += 1;
    }
    println!("Arc+Mutex = {:?}", a.lock().unwrap());
}
```

---

## 7. Concurrency (Threads, Mutex, Condvar)

### Key Points
- `thread::spawn` spawns a new thread.
- `Arc<Mutex<T>>` to share mutable state.
- `Condvar` for signaling between threads.
- `RwLock` → multiple readers or one writer.

### Example
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn thread_demo() {
    let counter = Arc::new(Mutex::new(0));

    let mut handles = vec![];
    for _ in 0..5 {
        let c = Arc::clone(&counter);
        let h = thread::spawn(move || {
            let mut num = c.lock().unwrap();
            *num += 1;
        });
        handles.push(h);
    }

    for h in handles { h.join().unwrap(); }
    println!("Counter = {}", *counter.lock().unwrap());
}
```

---

## 8. Channels (mpsc)

### Key Points
- `mpsc`: multi-producer, single-consumer.
- `tx.send()` / `rx.recv()`.

### Example
```rust
use std::sync::mpsc;
use std::thread;

fn channel_demo() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send("hello").unwrap();
    });

    println!("Got: {}", rx.recv().unwrap());
}
```

---

## 9. Async & Tokio

### Key Points
- `async fn` returns `Future`.
- Needs executor (tokio).
- `tokio::spawn` for lightweight tasks.

### Example
```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let h = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });
    println!("Result = {}", h.await.unwrap());
}
```

---

## 10. Iterators, Closures, Function Pointers

### Example
```rust
fn iter_demo() {
    let v = vec![1, 2, 3];
    let doubled: Vec<_> = v.iter().map(|x| x * 2).collect();
    println!("{:?}", doubled);

    let add = |a: i32, b: i32| a + b;
    println!("Sum = {}", add(2, 3));

    let f: fn(i32) -> i32 = |x| x * x;
    println!("Square = {}", f(4));
}
```

---

## 11. Macros

### Example
```rust
macro_rules! say_hello {
    () => {
        println!("Hello from macro!");
    };
}

fn main() {
    say_hello!();
}
```

---

## 12. Error Handling

### Example
```rust
fn might_fail(x: i32) -> Result<i32, String> {
    if x > 0 { Ok(x) } else { Err("negative!".into()) }
}

fn main() {
    match might_fail(-1) {
        Ok(v) => println!("Value = {}", v),
        Err(e) => println!("Error: {}", e),
    }
}
```

---

## 13. Unsafe, FFI & Static

### Example
```rust
static mut COUNTER: i32 = 0;

unsafe fn increment() {
    COUNTER += 1;
}

extern "C" {
    fn puts(s: *const i8);
}
```

---

## 14. Storage Classes (C vs Rust)

- `global` → module-level `static`.
- `static` → single instance.
- `extern` → link to external symbols.
- `register` → no direct equivalent.
- `thread_local!` macro for thread-local vars.

### Example
```rust
use std::cell::RefCell;

thread_local! {
    static TL: RefCell<i32> = RefCell::new(0);
}
```

---

## 15. Design Patterns

- **Singleton** → `lazy_static!` or `OnceCell`.
- **Builder** → chained methods.
- **Observer** → channels.
- **Strategy** → traits + dyn dispatch.
- **State** → enums + match.

### Example (Builder)
```rust
struct Config { a: i32, b: i32 }

impl Config {
    fn new() -> Self { Config { a: 0, b: 0 } }
    fn a(mut self, v: i32) -> Self { self.a = v; self }
    fn b(mut self, v: i32) -> Self { self.b = v; self }
}

fn main() {
    let c = Config::new().a(1).b(2);
    println!("Config: a={}, b={}", c.a, c.b);
}
```

---

## 16. Common Interview Snippets

```rust
// Reverse loop
for i in (1..=5).rev() { println!("{}", i); }

// Command line args
let args: Vec<String> = std::env::args().collect();

// Thread with name
let handle = std::thread::Builder::new()
    .name("worker".into())
    .spawn(|| println!("hi")).unwrap();

// Atomic counter
use std::sync::atomic::{AtomicUsize, Ordering};
static CNT: AtomicUsize = AtomicUsize::new(0);
CNT.fetch_add(1, Ordering::SeqCst);
```

---

# END
