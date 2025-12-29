# Chapter 11: Chains of Thought (Monads)

In this chapter, we generalize the patterns we've seen in `TestCaseGenerator` (Gen), `Extractor` (Parser), `Option`, and `Agent` (State). We identify two key abstractions for Agentic Flow Control: the **Functor** (Mapping) and the **Monad** (Chaining).

In LLM engineering, the **Chain of Thought** (CoT) pattern is ubiquitous. An agent thinks `A`, then uses `A` to think `B`. This dependency (`A -> B`) is exactly what a Monad captures.

## 11.1 Functors (Mapping Thoughts)

A Functor is a type `F` that provides a `map` function. E.g., `Option::map` transforms a potential value without handling the "None" case explicitly.

To work around Rust's lack of Higher-Kinded Types (HKTs), we use **Generic Associated Types (GATs)**.

```rust
pub trait Functor {
    type Wrapped<A>;
    
    fn map<A, B, F>(fa: Self::Wrapped<A>, f: F) -> Self::Wrapped<B> 
    where F: Fn(A) -> B;
}
# fn main() {}
```

## 11.2 Monads (Reasoning Chains)

The Monad abstraction adds `unit` (Thought) and `flat_map` (Next Step).
In Agent terms, `flat_map` allows the *next step* of the chain to be determined by the *result* of the previous step.

```rust
# pub trait Functor { type Wrapped<A>; fn map<A, B, F>(fa: Self::Wrapped<A>, f: F) -> Self::Wrapped<B> where F: Fn(A) -> B; }
pub trait Monad: Functor {
    fn unit<A>(a: A) -> Self::Wrapped<A>;
    
    fn flat_map<A, B, F>(ma: Self::Wrapped<A>, f: F) -> Self::Wrapped<B>
    where F: Fn(A) -> Self::Wrapped<B>;
}
# fn main() {}
```

## 11.3 Instances

We can implement Monad instances for:

*   **Option (Validation)**: Simple sequencing. Failure short-circuits the reasoning chain.
*   **Vec (Branching)**: Represents non-determinism (Generating multiple hypotheses). `flat_map` explores all branches.
*   **Id (Direct)**: The identity monad.
*   **Agent (State)**: The "Agent Monad" from Chapter 6. `flat_map` sequences actions while passing memory.

## 11.4 Combinators

Once we have the `Monad` trait, we can define powerful combinators that work for *any* reasoning chain:

*   **`sequence`**: Turns a list of steps `Vec<Step<A>>` into a single Step yielding a list `Step<Vec<A>>`.
*   **`filterM`**: Filters a list based on a reasoning predicate (e.g. "Keep only relevant documents").

## 11.5 Conclusion

Monads provide a unified interface for sequencing operations. They are the mathematical foundation of "Chain of Thought" prompting.

## 11.6 References

*   **Rust Blog (GATs)**: [Stabilization Announcement](https://blog.rust-lang.org/2022/10/28/gats-stabilization.html)
*   **Rust By Example (Traits)**: [Traits](https://doc.rust-lang.org/rust-by-example/trait.html)
