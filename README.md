# Rust Master Interview Cheat-Sheet with Layman Examples

This cheat-sheet explains key Rust concepts with simple, beginner-friendly examples to help you understand and prepare for interviews.

## 1. Basics

### Ownership & Borrowing
Rust ensures memory safety by having only one owner for data, but you can "borrow" it. Borrowing can be immutable (read-only, multiple allowed) or mutable (read-write, only one allowed).

**Example**: Think of a book. Only one person owns it, but many can read it (immutable borrow). Only one person can write in it at a time (mutable borrow).

```rust
fn main() {
    let mut book = String::from("Rust Guide"); // book is owned
    let reader1 = &book; // immutable borrow
    let reader2 = &book; // another immutable borrow
    println!("Readers: {}, {}", reader1, reader2);
    // let writer = &mut book; // Error! Can't borrow mutably while immutable borrows exist
}
```

### Lifetimes
Lifetimes ensure borrowed data isn't used after the owner is gone, preventing "dangling" references.

**Example**: Imagine borrowing a friend's phone. If they leave (data is dropped), you can't use the phone anymore. Lifetimes track this.

```rust
fn longest<'a>(s1: &'a str, s2: &'a str) -> &'a str { // 'a ensures return value lives as long as inputs
    if s1.len() > s2.len() { s1 } else { s2 }
}
fn main() {
    let s1 = String::from("short");
    let s2 = String::from("longer");
    let result = longest(&s1, &s2);
    println!("Longest: {}", result); // Works because s1 and s2 are still alive
}
```

### Storage Classes
Rust’s storage classes differ from C++. Here’s how they work:

- **Global (module-level)**: Like a shared library book, accessible everywhere, defined with `static` or `const`.
- **Static**: Single instance with a `'static` lifetime (lives for the entire program).
- **Extern**: For using code from other languages (e.g., C functions).
- **Register**: Rust doesn’t use this; the compiler optimizes automatically.

**Example**: A library’s shared counter (`static`) vs. a fixed math constant (`const`).

```rust
static LIBRARY_BOOKS: i32 = 100; // Single, shared value
const PI: f64 = 3.14159; // Fixed constant
fn main() {
    println!("Books in library: {}", LIBRARY_BOOKS);
    println!("Pi value: {}", PI);
}
```

### Command Line Args
Access command-line arguments to customize program behavior.

**Example**: Imagine a program as a waiter taking your order from the command line.

```rust
fn main() {
    for arg in std::env::args() { // Gets arguments like "cargo run pizza burger"
        println!("Order: {}", arg); // Prints program name, then "pizza", "burger"
    }
}
```

## 2. Concurrency

### Arc + Mutex
`Arc` (Atomic Reference Counting) allows multiple threads to share data safely, and `Mutex` ensures only one thread modifies it at a time.

**Example**: A shared cookie jar. Multiple people (threads) can look at it (`Arc`), but only one can take a cookie (`Mutex`).

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let jar = Arc::new(Mutex::new(10)); // 10 cookies
    let mut handles = vec![];

    for _ in 0..3 {
        let jar = Arc::clone(&jar);
        let handle = thread::spawn(move || {
            let mut cookies = jar.lock().unwrap();
            *cookies -= 1; // Take one cookie
            println!("Cookies left: {}", *cookies);
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

### Condvar
`Condvar` (Condition Variable) helps threads wait for a signal to proceed.

**Example**: A chef waits for ingredients to arrive before cooking.

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = Arc::clone(&pair);

    thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ingredients = lock.lock().unwrap();
        *ingredients = true; // Ingredients arrived
        cvar.notify_one(); // Signal the chef
    });

    let (lock, cvar) = &*pair;
    let mut ingredients = lock.lock().unwrap();
    while !*ingredients {
        ingredients = cvar.wait(ingredients).unwrap(); // Wait for ingredients
    }
    println!("Chef starts cooking!");
}
```

### notify_one vs notify_all
`notify_one` wakes one waiting thread; `notify_all` wakes all waiting threads.

**Example**: `notify_one` is like calling one friend to join a party; `notify_all` is like announcing to everyone.

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(0), Condvar::new()));
    let pair2 = Arc::clone(&pair);

    for _ in 0..2 {
        let pair = Arc::clone(&pair2);
        thread::spawn(move || {
            let (lock, cvar) = &*pair;
            let mut count = lock.lock().unwrap();
            *count += 1;
            cvar.notify_all(); // Wake all waiting threads
            println!("Thread done, count: {}", *count);
        });
    }
}
```

### Thread Starvation
Starvation happens when a thread can’t get the resources it needs because others keep taking them.

**Example**: At a buffet, one person keeps missing food because others grab it first.

### Round-Robin with Condvar
A round-robin scheduler lets tasks take turns using a condition variable.

**Example**: Four friends take turns playing a game, signaling the next player.

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::thread;
use std::time::Duration;

fn main() {
    let pair = Arc::new((Mutex::new(0), Condvar::new()));
    let mut handles = vec![];

    for i in 0..4 {
        let pair = Arc::clone(&pair);
        let handle = thread::spawn(move || {
            let (lock, cvar) = &*pair;
            loop {
                let mut turn = lock.lock().unwrap();
                if *turn % 4 == i {
                    println!("Player {}'s turn", i);
                    *turn += 1;
                    cvar.notify_all();
                }
                turn = cvar.wait_timeout(turn, Duration::from_millis(100)).unwrap().0;
            }
        });
        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

## 3. Async / Await

### tokio::spawn
`tokio::spawn` creates lightweight async tasks.

**Example**: Like assigning multiple waiters to serve tables without waiting for each to finish.

```rust
use tokio::task;

#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        println!("Waiter 1 serving...");
    });
    println!("Main keeps running!");
    handle.await.unwrap();
}
```

### Don’t Block with thread::sleep
Use `tokio::time::sleep` in async code to avoid blocking other tasks.

**Example**: A waiter pauses politely (async sleep) instead of freezing the restaurant (thread sleep).

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("Start");
    sleep(Duration::from_secs(1)).await; // Polite pause
    println!("After 1 second");
}
```

### Async vs Threads
Async tasks cooperate and share time; threads run independently and can be preempted.

**Example**: Async is like waiters taking turns serving; threads are like chefs cooking at the same time.

## 4. Traits & Polymorphism

### Traits
Traits define shared behavior, like interfaces in other languages.

**Example**: A trait for animals that can make sounds.

```rust
trait Animal {
    fn make_sound(&self);
}

struct Dog;
struct Cat;

impl Animal for Dog {
    fn make_sound(&self) {
        println!("Woof!");
    }
}

impl Animal for Cat {
    fn make_sound(&self) {
        println!("Meow!");
    }
}

fn main() {
    let dog = Dog;
    let cat = Cat;
    dog.make_sound();
    cat.make_sound();
}
```

### Pure Virtual-Like Methods
Traits can define methods without implementation, like pure virtual functions.

**Example**: An animal must make a sound, but each defines how.

```rust
trait Animal {
    fn make_sound(&self); // Must be implemented
}

struct Bird;
impl Animal for Bird {
    fn make_sound(&self) {
        println!("Chirp!");
    }
}
```

### Default Methods
Traits can have default implementations.

**Example**: Animals can have a default way to sleep.

```rust
trait Animal {
    fn sleep(&self) {
        println!("Zzz...");
    }
}

struct Dog;
impl Animal for Dog {}

fn main() {
    let dog = Dog;
    dog.sleep(); // Uses default implementation
}
```

### Trait Inheritance
A trait can require another trait.

**Example**: A pet must be an animal first.

```rust
trait Animal {
    fn eat(&self);
}

trait Pet: Animal {
    fn play(&self);
}

struct Dog;
impl Animal for Dog {
    fn eat(&self) {
        println!("Dog eats bone");
    }
}
impl Pet for Dog {
    fn play(&self) {
        println!("Dog plays fetch");
    }
}
```

### Polymorphism with Vec<Box<dyn Trait>>
Store different types implementing the same trait.

**Example**: A zoo with different animals.

```rust
trait Animal {
    fn make_sound(&self);
}

struct Dog;
struct Cat;

impl Animal for Dog {
    fn make_sound(&self) { println!("Woof!"); }
}
impl Animal for Cat {
    fn make_sound(&self) { println!("Meow!"); }
}

fn main() {
    let zoo: Vec<Box<dyn Animal>> = vec![Box::new(Dog), Box::new(Cat)];
    for animal in zoo {
        animal.make_sound();
    }
}
```

### Constructors
Use `fn new() -> Self` for creating instances.

**Example**: A car factory.

```rust
struct Car {
    model: String,
}

impl Car {
    fn new(model: String) -> Self {
        Car { model }
    }
}

fn main() {
    let car = Car::new(String::from("Sedan"));
    println!("Car model: {}", car.model);
}
```

### Destructors (Drop Trait)
The `Drop` trait runs cleanup when an object goes out of scope.

**Example**: A file that closes itself when done.

```rust
struct File {
    name: String,
}

impl Drop for File {
    fn drop(&mut self) {
        println!("Closing file: {}", self.name);
    }
}

fn main() {
    let _file = File { name: String::from("data.txt") };
    // File is closed automatically when scope ends
}
```

## 5. Memory & Abstractions

### Zero-Cost Abstractions
Rust’s high-level code (like iterators) compiles to efficient machine code, like raw loops.

**Example**: Using an iterator is as fast as a for loop.

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    let sum: i32 = numbers.iter().sum(); // As fast as a manual loop
    println!("Sum: {}", sum);
}
```

### RAII
Resources are freed automatically when objects go out of scope.

**Example**: A file closes itself when you’re done.

```rust
struct File {
    name: String,
}

impl Drop for File {
    fn drop(&mut self) {
        println!("File {} closed", self.name);
    }
}

fn main() {
    let _file = File { name: String::from("log.txt") };
    println!("Working with file");
    // File closes automatically
}
```

### Static Keyword
`static` for single-instance variables, `const` for compile-time constants.

**Example**: A counter for visitors vs. a fixed tax rate.

```rust
static mut VISITORS: i32 = 0; // Shared counter
const TAX_RATE: f64 = 0.08; // Fixed constant

fn main() {
    unsafe {
        VISITORS += 1; // Unsafe because static mut is dangerous
        println!("Visitors: {}", VISITORS);
    }
    println!("Tax rate: {}", TAX_RATE);
}
```

## 6. Macros

### Declarative Macros
Simple macros for pattern matching.

**Example**: A macro to say hello.

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}

fn main() {
    say_hello!(); // Prints "Hello!"
}
```

### Procedural Macros
Advanced macros like `#[derive(Debug)]`.

**Example**: Auto-generate debug formatting.

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p); // Prints "Point { x: 1, y: 2 }"
}
```

## 7. Patterns / Design Patterns

### Singleton
Use `lazy_static` or `once_cell::Lazy` for a single instance.

**Example**: One global configuration.

```rust
use once_cell::sync::Lazy;

static CONFIG: Lazy<i32> = Lazy::new(|| 42);

fn main() {
    println!("Config: {}", *CONFIG);
}
```

### Builder
Chain methods to build objects.

**Example**: Build a sandwich step by step.

```rust
struct Sandwich {
    bread: String,
    filling: String,
}

impl Sandwich {
    fn new() -> SandwichBuilder {
        SandwichBuilder {
            bread: None,
            filling: None,
        }
    }
}

struct SandwichBuilder {
    bread: Option<String>,
    filling: Option<String>,
}

impl SandwichBuilder {
    fn bread(mut self, bread: &str) -> Self {
        self.bread = Some(bread.to_string());
        self
    }
    fn filling(mut self, filling: &str) -> Self {
        self.filling = Some(filling.to_string());
        self
    }
    fn build(self) -> Sandwich {
        Sandwich {
            bread: self.bread.unwrap_or("White".to_string()),
            filling: self.filling.unwrap_or("Cheese".to_string()),
        }
    }
}

fn main() {
    let sandwich = Sandwich::new()
        .bread("Rye")
        .filling("Ham")
        .build();
    println!("Sandwich: {} with {}", sandwich.bread, sandwich.filling);
}
```

### Strategy
Use traits for interchangeable algorithms.

**Example**: Different ways to travel.

```rust
trait Travel {
    fn go(&self);
}

struct Car;
struct Bike;

impl Travel for Car {
    fn go(&self) {
        println!("Driving a car");
    }
}
impl Travel for Bike {
    fn go(&self) {
        println!("Riding a bike");
    }
}

fn main() {
    let transport: Box<dyn Travel> = Box::new(Car);
    transport.go();
}
```

### Observer
Use channels (`mpsc`) to notify observers.

**Example**: A weather station notifies subscribers.

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx2 = tx.clone();

    thread::spawn(move || {
        tx2.send("Rainy").unwrap();
    });

    println!("Weather update: {}", rx.recv().unwrap());
}
```

### RAII
Resources are managed via scope, using `Drop`.

**Example**: A resource that cleans itself up.

```rust
struct Resource {
    name: String,
}

impl Drop for Resource {
    fn drop(&mut self) {
        println!("Cleaning up {}", self.name);
    }
}

fn main() {
    let _r = Resource { name: String::from("Database") };
    println!("Using resource");
}
```

### Smart Pointers
`Box`, `Rc`, `Arc` manage memory.

**Example**: A shared book with `Rc`.

```rust
use std::rc::Rc;

fn main() {
    let book = Rc::new(String::from("Rust Book"));
    let reader1 = Rc::clone(&book);
    let reader2 = Rc::clone(&book);
    println!("Readers have: {}", book);
}
```

### State Pattern
Use enums and `match` for state transitions.

**Example**: A traffic light.

```rust
enum TrafficLight {
    Red,
    Yellow,
    Green,
}

impl TrafficLight {
    fn next(&self) -> TrafficLight {
        match self {
            TrafficLight::Red => TrafficLight::Yellow,
            TrafficLight::Yellow => TrafficLight::Green,
            TrafficLight::Green => TrafficLight::Red,
        }
    }
}

fn main() {
    let mut light = TrafficLight::Red;
    light = light.next();
    match light {
        TrafficLight::Yellow => println!("Yellow light"),
        _ => println!("Other light"),
    }
}
```

## 8. Extras / Misc

### Modules & Visibility
Use `pub` for public items, private by default.

**Example**: A library with private and public books.

```rust
mod library {
    pub fn public_book() {
        println!("Public book");
    }
    fn private_book() {
        println!("Private book");
    }
}

fn main() {
    library::public_book();
    // library::private_book(); // Error: private
}
```

### Error Handling
Use `Result` and `Option` for safe error handling.

**Example**: Try to open a file.

```rust
use std::fs::File;

fn main() -> Result<(), std::io::Error> {
    let _file = File::open("data.txt")?; // ? propagates error
    println!("File opened");
    Ok(())
}
```

### Function Pointers
Store functions in variables.

**Example**: A calculator function.

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    let f: fn(i32, i32) -> i32 = add;
    println!("Sum: {}", f(2, 3));
}
```

### Reverse Loop
Iterate in reverse with `.rev()`.

**Example**: Count down like a rocket launch.

```rust
fn main() {
    for i in (1..=5).rev() {
        println!("Countdown: {}", i);
    }
}
```
