# Chapter 11: Monads

In this chapter, we generalize the patterns we've seen in `Gen`, `Parser`, `Option`, and `State`. We identify two key abstractions: the **Functor** and the **Monad**.

## 11.1 Functors

A Functor is a type constructor `F` that provides a `map` function. In Scala, this is represented as `trait Functor[F[_]]`. In Rust, we lack native support for Higher-Kinded Types (HKTs) (i.e., types that take other types as type arguments, like `Option` itself, not `Option<i32>`).

To work around this, we use **Generic Associated Types (GATs)**, a feature stabilized in Rust 1.65. We define a trait that has an associated type `Wrapped<A>` representing `F<A>`.

> [!NOTE]
> Learn more about GATs in the [Official Rust Blog Post](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html).

```rust
pub trait Functor {
    type Wrapped<A>;
    
    fn map<A, B, F>(fa: Self::Wrapped<A>, f: F) -> Self::Wrapped<B> 
    where F: Fn(A) -> B;
}
```

This allows us to write generic code that works for any Functor, provided we implement the trait for a specific "Monad Type" (like `OptionMonad` acts as the descriptor for `Option`).

## 11.2 Monads

The Monad abstraction adds `unit` and `flatMap` (often called `bind` or `>>=` in other languages).

Make note that **Rust's standard library** uses the term `and_then` for `Option` and `Result`, and `flat_map` for `Iterator`.

```rust
pub trait Monad: Functor {
    fn unit<A>(a: A) -> Self::Wrapped<A>;
    
    fn flat_map<A, B, F>(ma: Self::Wrapped<A>, f: F) -> Self::Wrapped<B>
    where F: Fn(A) -> Self::Wrapped<B>;
    
    // ... map2 ...
}
```

### The `map2` Challenge in Rust

In functional theory, `map2` can be derived from `flatMap` and `map`. However, in Rust, implementing `map2` generically via `flatMap` involves closures.

```rust
// Conceptual default implementation
flat_map(ma, |a| map(mb, |b| f(a, b)))
```

This inner closure `|b| f(a, b)` captures `a` and `f`. If the Monad executes this closure multiple times (like `Vec`/List monad, which handles non-determinism), `f` and `a` must be clonable. Because of this complexity in a generic context, we essentially require `map2` to be implemented (or specialized) by the instance, or we impose `Clone` bounds on the function `F` and the values.

## 11.3 Instances

We implemented Monad instances for:

*   **Option**: Simple sequencing. Failure short-circuits. Equivalent to `Option::and_then`.
*   **Vec (List)**: Represents non-determinism. `flatMap` applies the function to each element and concatenates the results. Equivalent to `Iterator::flat_map`.
*   **Id**: The identity monad, wrapping a value directly.
*   **Result**: Similar to Option but carries an error value. Equivalent to `Result::and_then`.

## 11.4 Combinators

Once we have the `Monad` trait, we can define powerful combinators that work for *any* monad:

*   **`sequence`**: Turns a `Vec<M<A>>` into `M<Vec<A>>`.
*   **`traverse`**: Maps a function `A -> M<B>` over a list and collects the results.
*   **`replicateM`**: Generates a list of `n` values from sequencing a monad.
*   **`filterM`**: A powerful combinator that filters a list based on a monadic predicate. For `Vec` (List Monad), this generates the powerset (all subsequences) of a list!

## 11.5 Conclusion

Monads provide a unified interface for sequencing operations. Despite Rust's type system differences from Scala (lack of HKTs), we can effectively model these concepts using GATs, allowing us to write reusable, generic control flow logic.

## 11.6 References

*   **Rust Blog (GATs)**: [Stabilization Announcement](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html)
*   **Rust By Example (Traits)**: [Traits](https://doc.rust-lang.org/rust-by-example/trait.html)
