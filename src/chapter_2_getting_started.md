# Chapter 2: Getting started with functional programming in Rust

In this chapter, we’ll begin learning how to write programs in the Rust language just by combining pure functions. This chapter is mainly intended for those readers who are new to Rust, to functional programming, or both.

## 2.1 Introducing Rust: an example

The following is a complete program listing in Rust. Our goal is just to introduce the Rust language and its syntax.

```rust
// A comment!
/* Another comment */
/// A documentation comment

pub mod my_module {
    pub fn abs(n: i32) -> i32 {
        if n < 0 {
            -n
        } else {
            n
        }
    }

    pub fn format_abs(x: i32) -> String {
        format!("The absolute value of {} is {}", x, abs(x))
    }

    pub fn main() {
        println!("{}", format_abs(-42));
    }
}
# fn main() {
#     my_module::main();
# }
```

We declare a module `my_module`. This is simply to give our code a place to live. Rust code is organized into modules (`mod`) and creates (structs, enums).

The `abs` function is a pure function that takes an integer and returns its absolute value. Note the absence of an explicit `return` keyword. The value returned from a function is simply whatever value results from evaluating the last expression in the block.

## 2.2 Running our program

The simplest way to run this is using `cargo`. If this were in `src/main.rs`, you could run:

```sh
cargo run
```

## 2.3 Higher-order functions: passing functions to functions

Functions are values. They can be assigned to variables, stored in data structures, and passed as arguments to functions.

### 2.3.1 A short detour: writing loops functionally

First, let’s write `factorial`:

```rust
fn factorial(n: i32) -> i32 {
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
#     assert_eq!(factorial(5), 120);
# }
```

The way we write loops functionally, without mutating a loop variable, is with a recursive function. Rust supports recursion, though it does not guarantee **tail call elimination** (TCO) in the same way Scala or Scheme might. For deep recursion, Rust programmers often use `loop`, `while`, or iterators to avoid blowing the stack. However, for the sake of learning FP concepts, we will use recursion here.

### Exercise 2.1
Write a recursive function to get the `n`th Fibonacci number. The first two Fibonacci numbers are 0 and 1. The `n`th number is always the sum of the previous two.

```rust
pub fn fib(n: u32) -> u32 {
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
#     assert_eq!(fib(0), 0);
#     assert_eq!(fib(1), 1);
#     assert_eq!(fib(5), 5);
#     assert_eq!(fib(6), 8);
# }
```

### 2.3.2 Writing our first higher-order function

We can generalize `format_abs` and `format_factorial` to a single function `format_result`:

```rust
fn format_result(name: &str, n: i32, f: fn(i32) -> i32) -> String {
    format!("The {} of {} is {}.", name, n, f(n))
}
# fn main() {
#     fn double(x: i32) -> i32 { x * 2 }
#     println!("{}", format_result("double", 21, double));
# }
```

`f` is a function pointer or closure trait. Here we use `fn(i32) -> i32`, which is a function pointer.

## 2.4 Polymorphic functions: abstracting over types

We can write functions that work for any type.

### 2.4.1 An example of a polymorphic function

```rust
fn find_first<A, F>(as_slice: &[A], p: F) -> Option<usize> 
where F: Fn(&A) -> bool {
    let mut n = 0;
    while n < as_slice.len() {
        if p(&as_slice[n]) {
            return Some(n);
        }
        n += 1;
    }
    None
}
# fn main() {
#     let data = vec![1, 2, 3, 4, 5];
#     assert_eq!(find_first(&data, |x| *x == 3), Some(2));
# }
```

Here, `A` is a generic type parameter. `F` is a generic type bounded by the `Fn` trait.

### Exercise 2.2
Implement `is_sorted`, which checks whether a slice `&[A]` is sorted according to a given comparison function.

```rust
pub fn is_sorted<A, F>(as_slice: &[A], ordered: F) -> bool 
where F: Fn(&A, &A) -> bool {
    let mut n = 0;
    while n + 1 < as_slice.len() {
        if !ordered(&as_slice[n], &as_slice[n+1]) {
            return false;
        }
        n += 1;
    }
    true
}
# fn main() {
#     assert!(is_sorted(&[1, 2, 3], |a, b| a <= b));
#     assert!(!is_sorted(&[1, 3, 2], |a, b| a <= b));
# }
```

## 2.5 Following types to implementations

### Exercise 2.3 (Currying)
Currying converts a function `f` of two arguments into a function of one argument that partially applies `f`.
Note: In Rust, returning closures involving references or generics can be complex due to lifetimes. We use `impl Fn` (existential type) to return a closure.

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
Implement the higher-order function that composes two functions.

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
We introduced Rust syntax, recursion, higher-order functions (HOFs), and polymorphism.

## 2.7 References

*   **Functions**: [The Rust Book Ch 3.3](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html)
*   **Generics (`<T>`)**: [The Rust Book Ch 10.1](https://doc.rust-lang.org/book/ch10-01-syntax.html)
*   **Trait Bounds (`where F: Fn`)**: [The Rust Book Ch 10.2](https://doc.rust-lang.org/book/ch10-02-traits.html)
*   **Closures & Iterators**: [The Rust Book Ch 13](https://doc.rust-lang.org/book/ch13-00-functional-features.html)
*   **Rust By Example (Functions)**: [Functions](https://doc.rust-lang.org/rust-by-example/fn.html)
*   **Rust By Example (Closures)**: [Closures](https://doc.rust-lang.org/rust-by-example/fn/closures.html)
