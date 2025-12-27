# Chapter 14: Local Effects and Mutable State

In functional programming, we emphasize immutability. However, some algorithms (like Quicksort, HashMaps, or graph traversals) are naturally expressed with mutable state. The types `ST` (State Thread) and `STRef` allow us to use mutable state *locally* while remaining externally pure.

## 14.1 Purely Functional In-Place Mutation?

It sounds like a contradiction. How can we have mutation in a pure function?
The key is **Scope**.
If a function allocates a mutable array, modifies it, and then returns a frozen (immutable) copy (or a result derived from it), and *no one else* ever saw the mutable version, then the function is pure from the outside.

## 14.2 The ST Monad in Rust

We implemented `ST<'a, A>` where `'a` represents the "thread" or scope of the mutation.

```rust
pub struct ST<'a, A> {
    run: Box<dyn FnOnce() -> A + 'a>, // Closure capturing mutable 'a references
}
```

### Preventing Leakage
The danger is leaking a mutable reference:
```rust
let r = run_st(STRef::new(1)); // ERROR! Result cannot be STRef
```

In Rust, we prevent this using Higher-Rank Trait Bounds (HRTB) in the `run_st` function:

```rust
pub trait RunnableST<A> {
    fn apply<'a>(&self) -> ST<'a, A>;
}

pub fn run_st<A>(st: &impl RunnableST<A>) -> A { ... }
```

Because `run_st` enforces that `RunnableST` works for *any* `'a` (`for<'a>`), the return type `A` cannot depend on `'a`. `STRef<'a, T>` depends on `'a`, so it cannot be returned. `Vec<T>` (via `freeze()`) does *not* depend on `'a`, so it is safe to return.

## 14.3 Comparison with Rust References

Rust's borrow checker provides similar guarantees natively:
- `&mut T` ensures exclusive access (effectively a local state thread).
- Lifetimes (`'a`) ensure references don't escape their owner.

In a sense, **Rust is the ST Monad**. Every block of code with local variables is an implicit `ST` computation. However, the explicit `ST` monad is useful when we want to treat "stateful computations" as First-Class Values (e.g., to pass them around, combine them safely, or implement "Ghost Types" patterns).

## 14.4 Conclusion

We now have the ability to write efficient, in-place algorithms (like the `partition` step of Quicksort) using the `ST` monad, wrapping them safely so they appear pure to the rest of the program.

## 14.5 Rust Equivalents

*   **Smart Pointers**: [The Rust Book Ch 15](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
*   **Interior Mutability (`RefCell`)**: [The Rust Book Ch 15.5](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)
*   **Lifetimes**: [The Rust Book Ch 10.3](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html)
