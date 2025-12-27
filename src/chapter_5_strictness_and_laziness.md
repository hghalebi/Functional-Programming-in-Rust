# Chapter 5: Strictness and Laziness

In this chapter, we explore **non-strictness** (laziness) to improve efficiency and modularity. We implement a `Stream` type that fuses sequences of transformations.

## 5.1 Strict and non-strict functions

Strict functions evaluate their arguments *before* the function body is executed. Non-strict functions may choose not to evaluate arguments.

In Rust, arguments are strictly evaluated by default. To simulate non-strictness, we can pass a closure (thunk) `Fn() -> A` instead of a value `A`.

```rust
// logical AND is non-strict in the second argument
// false && { println!("!!"); true } // does not print
```

## 5.2 An extended example: lazy lists

We define a `Stream` (lazy list). In Scala, `Cons` takes by-name arguments. In Rust, we use closures wrapped in `Rc` (for sharing) to represent these thunks.

```rust
use std::rc::Rc;

#[derive(Clone)]
pub enum Stream<A> {
    Empty,
    Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> Stream<A>>),
}
// Note: We avoid memoization complexity for this basic translation. 
// In a production Rust persistent stream, one might use `lazy_static` or `OnceCell` inside `Rc`.
```

### Exercise 5.1: to_list
Force the stream into a strict `List` (or `Vec`).

```rust
// impl Stream<A>
pub fn to_vec(&self) -> Vec<A> {
    match self {
        Stream::Empty => Vec::new(),
        Stream::Cons(h, t) => {
            let mut v = vec![h()];
            v.extend(t().to_vec());
            v
        }
    }
}
```

### Exercise 5.2: take and drop
```rust
pub fn take(&self, n: usize) -> Stream<A> {
    if n == 0 {
        Stream::Empty
    } else {
        match self {
            Stream::Empty => Stream::Empty,
            Stream::Cons(h, t) => {
                let h = h.clone();
                let t = t.clone();
                Stream::cons(move || h(), move || t().take(n - 1))
            }
        }
    }
}
```

### Exercise 5.3: take_while
Return all starting elements that match a predicate.

## 5.3 Separating program description from evaluation

Laziness lets us separate the description of an expression from its evaluation.

### Exercise 5.4: for_all
Check that all elements in the Stream match a given predicate, terminating early if possible.

### Exercise 5.5: take_while using fold_right
### Exercise 5.6: head_option using fold_right
### Exercise 5.7: map, filter, append, flat_map using fold_right

## 5.4 Infinite streams and corecursion

Because functions are incremental, they work for infinite streams.

```rust
pub fn ones() -> Stream<i32> {
    Stream::cons(|| 1, ones)
}
```

### Exercise 5.8: constant
### Exercise 5.9: from
### Exercise 5.10: fibs
### Exercise 5.11: unfold
A general stream-building function (corecursion).

```rust
pub fn unfold<A, S, F>(z: S, f: F) -> Stream<A>
where F: Fn(S) -> Option<(A, S)> + 'static + Clone {
    match f(z) {
        Some((a, s)) => {
            let f = f.clone();
            Stream::cons(move || a.clone(), move || unfold(s, f.clone()))
        },
        None => Stream::Empty,
    }
}
```

### Exercise 5.12: fibs, from, constant, ones via unfold
### Exercise 5.13: map, take, take_while, zip_with, zip_all via unfold
### Exercise 5.14: starts_with
### Exercise 5.15: tails
### Exercise 5.16: scan_right

## 5.5 Summary
Laziness improves modularity by decoupling description from evaluation.

## 5.6 References

*   **Rust Book Ch 13 (Iterators)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch13-02-iterators.html)
*   **Rust By Example (Iterators)**: [Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html)
*   **Rust Patterns (Lazy Evaluation)**: [Idioms](https://rust-unofficial.github.io/patterns/idioms/lazy-evaluation.html)
