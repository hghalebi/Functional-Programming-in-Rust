# Chapter 2: Getting started with functional programming in Rust

In this chapter, we’ll begin learning how to write programs in the Rust language just by combining pure functions (Deterministic Tools). This chapter is mainly intended for those readers who are new to Rust, to functional programming, or both.

## 2.1 Introducing Rust: an example

The following is a complete program listing in Rust. Our goal is just to introduce the Rust language and its syntax.

```rust
// A comment!
/* Another comment */
/// A documentation comment

pub mod my_module {
    /// Pure function to calculate remaining tokens in a context window
    pub fn remaining_context(used: i32, limit: i32) -> i32 {
        let remaining = limit - used;
        if remaining < 0 {
            0
        } else {
            remaining
        }
    }

    pub fn format_status(used: i32) -> String {
        format!("Context used: {}, Remaining: {}", used, remaining_context(used, 8192))
    }

    pub fn main() {
        println!("{}", format_status(4000));
    }
}
# fn main() {
#     my_module::main();
# }
```

We declare a module `my_module`. This is simply to give our code a place to live. Rust code is organized into modules (`mod`) and creates (structs, enums).

The `remaining_context` function is a pure function that takes two integers and returns the remaining capacity. Note the absence of an explicit `return` keyword. The value returned from a function is simply whatever value results from evaluating the last expression in the block.

## 2.2 Running our program

The simplest way to run this is using `cargo`. If this were in `src/main.rs`, you could run:

```sh
cargo run
```

## 2.3 Higher-order functions: passing functions to functions

Functions are values. They can be assigned to variables, stored in data structures, and passed as arguments to functions.

### 2.3.1 A short detour: writing loops functionally

First, let’s write `retry_backoff` to calculate the wait time (in ms) for the `n`th retry of a failed API call (simple factorial-style growth for demonstration):

```rust
fn retry_backoff(n: i32) -> i32 {
    fn go(n: i32, acc: i32) -> i32 {
        if n <= 0 {
            acc
        } else {
            go(n - 1, n * acc)
        }
    }
    go(n, 1)
}
# fn main() {
#     assert_eq!(retry_backoff(5), 120);
# }
```

The way we write loops functionally, without mutating a loop variable, is with a recursive function. Rust supports recursion, though it does not guarantee **tail call elimination** (TCO) in the same way Scala or Scheme might. For deep recursion (like an infinite agent loop), Rust programmers often use `loop`, `while`, or iterators to avoid blowing the stack. However, for the sake of learning FP concepts, we will use recursion here.

### Exercise 2.1
Write a recursive function to simulate a simplified Fibonacci-style token generation curve. The 0th and 1st steps produce 0 and 1 tokens respectively. The `n`th step produces the sum of the previous two.

```rust
pub fn token_simulation(n: u32) -> u32 {
    fn go(n: u32, prev: u32, curr: u32) -> u32 {
        if n == 0 {
            prev
        } else {
            go(n - 1, curr, prev + curr)
        }
    }
    go(n, 0, 1)
}
# fn main() {
#     assert_eq!(token_simulation(0), 0);
#     assert_eq!(token_simulation(1), 1);
#     assert_eq!(token_simulation(5), 5);
#     assert_eq!(token_simulation(6), 8);
# }
```

### 2.3.2 Writing our first higher-order function

We can generalize `format_status` and `format_backoff` to a single function `format_metric`:

```rust
fn format_metric(name: &str, n: i32, f: fn(i32) -> i32) -> String {
    format!("The {} for input {} is {}.", name, n, f(n))
}
# fn main() {
#     fn double(x: i32) -> i32 { x * 2 }
#     println!("{}", format_metric("doubling", 21, double));
# }
```

`f` is a function pointer or closure trait. Here we use `fn(i32) -> i32`, which is a function pointer.

## 2.4 Polymorphic functions: abstracting over types

We can write functions that work for any type.

### 2.4.1 An example of a polymorphic function

Finding the first relevant Tool Call in a list of messages:

```rust
fn find_first_tool<A, F>(items: &[A], predicate: F) -> Option<usize> 
where F: Fn(&A) -> bool {
    let mut n = 0;
    while n < items.len() {
        if predicate(&items[n]) {
            return Some(n);
        }
        n += 1;
    }
    None
}
# fn main() {
#     let tools = vec!["search", "calculator", "weather"];
#     assert_eq!(find_first_tool(&tools, |x| *x == "calculator"), Some(1));
# }
```

Here, `A` is a generic type parameter. `F` is a generic type bounded by the `Fn` trait.

### Exercise 2.2
Implement `is_chronological`, which checks whether a slice of `Message`s `&[A]` is sorted according to a given comparison function (e.g., timestamps).

```rust
pub fn is_chronological<A, F>(messages: &[A], ordered: F) -> bool 
where F: Fn(&A, &A) -> bool {
    let mut n = 0;
    while n + 1 < messages.len() {
        if !ordered(&messages[n], &messages[n+1]) {
            return false;
        }
        n += 1;
    }
    true
}
# fn main() {
#     assert!(is_chronological(&[1, 2, 3], |a, b| a <= b));
#     assert!(!is_chronological(&[1, 3, 2], |a, b| a <= b));
# }
```

## 2.5 Following types to implementations

### Exercise 2.3 (Currying)
Currying converts a function `f` of two arguments (e.g., a function taking `(SystemPrompt, UserQuery)`) into a function of one argument that partially applies the first (e.g., returns a function waiting for `UserQuery` with `SystemPrompt` baked in).

```rust
pub fn curry<A, B, C, F>(f: F) -> impl Fn(A) -> Box<dyn Fn(B) -> C>
where
    A: 'static + Clone,
    B: 'static,
    C: 'static,
    F: Fn(A, B) -> C + 'static + Clone,
{
    move |a: A| {
        let f = f.clone();
        Box::new(move |b: B| f(a.clone(), b))
    }
}
# fn main() {
#     let add = |a, b| a + b;
#     let curried_add = curry(add);
#     let add_5 = curried_add(5);
#     assert_eq!(add_5(3), 8);
# }
``` 
*Note: This strictly follows the type signature but requires allocation (`Box`) and cloning to satisfy the borrow checker for arbitrary `A`, `B`. In idiomatic Rust, we rarely "curry" manually, but the concept stands.*

### Exercise 2.4 (Uncurry)
```rust
pub fn uncurry<A, B, C, F>(f: F) -> impl Fn(A, B) -> C
where
    F: Fn(A) -> Box<dyn Fn(B) -> C> + 'static,
{
    move |a: A, b: B| {
        let g = f(a);
        g(b)
    }
}
# fn main() {} // Usage requires curry to be useful test
```

### Exercise 2.5 (Compose)
Implement the higher-order function that composes two functions (e.g., compose `extract_json` and `execute_tool`).

```rust
pub fn compose<A, B, C, F, G>(f: F, g: G) -> impl Fn(A) -> C
where
    F: Fn(B) -> C + 'static,
    G: Fn(A) -> B + 'static,
    A: 'static,
{
    move |a: A| f(g(a))
}
# fn main() {
#    let f = |x| x + 1;
#    let g = |x| x * 2;
#    let h = compose(f, g);
#    assert_eq!(h(5), 11);
# }
```

## 2.6 Summary
We introduced Rust syntax, recursion, higher-order functions (Combined Tools), and polymorphism.

## 2.7 References

*   **Functions**: [The Rust Book Ch 3.3](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html)
*   **Generics (`<T>`)**: [The Rust Book Ch 10.1](https://doc.rust-lang.org/book/ch10-01-syntax.html)
*   **Trait Bounds (`where F: Fn`)**: [The Rust Book Ch 10.2](https://doc.rust-lang.org/book/ch10-02-traits.html)
*   **Closures & Iterators**: [The Rust Book Ch 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html)
*   **Rust By Example (Functions)**: [Functions](https://doc.rust-lang.org/rust-by-example/fn.html)
*   **Rust By Example (Closures)**: [Closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html)
