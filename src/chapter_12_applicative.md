# Chapter 12: Parallel Queries (Applicative and Traversable)

In this chapter, we discover that **Reasoning Chain** (Monad) is not the only useful abstraction for Agentic computation. We introduce **Parallel Executor** (Applicative), which allows for different behaviors, such as parallel tool execution and error accumulation. We then generalize the `traverse` and `sequence` functions into the **Batch Executor** (Traverse) trait.

## 12.1 Parallel Executors (Applicatives)

A Reasoning Chain (Monad) is defined by `unit` and `flat_map` (sequential).
A Parallel Executor (Applicative) is defined by `unit` and `map2` (parallel).

```rust
# pub trait Functor { type Wrapped<A>; fn map<A, B, F>(fa: Self::Wrapped<A>, f: F) -> Self::Wrapped<B> where F: Fn(A) -> B; }
pub trait Applicative: Functor {
    fn unit<A>(a: A) -> Self::Wrapped<A>;
    
    fn map2<A, B, C, F>(
        fa: Self::Wrapped<A>,
        fb: Self::Wrapped<B>,
        f: F
    ) -> Self::Wrapped<C>;
}
# fn main() {}
```

### Why Applicative?

If we look at `map2` vs `flat_map`:
- `flat_map` allows the *structure* of the second thought to depend on the *result* of the first. "If tool A returns X, call tool B. Else call C."
- `map2` takes two independent tasks and combines their results. "Call tool A and tool B."

This independence means:
1.  **Parallelism**: Since `fa` and `fb` are independent, they can be computed in parallel.
2.  **Analysis**: We can inspect the structure of the computation plan without running it.

## 12.2 Batch Executor (Traverse)

We've seen `batch_execute` (traverse) for Lists and Options. We can abstract this into a **Traverse** trait.

A Traversable functor allows us to iterate over a data structure (like a `ConversationTree`) while maintaining an effect (Parallel Execution) context.

```rust
# pub trait Functor { type Wrapped<A>; fn map<A, B, F>(fa: Self::Wrapped<A>, f: F) -> Self::Wrapped<B> where F: Fn(A) -> B; }
# pub trait Applicative: Functor { fn unit<A>(a: A) -> Self::Wrapped<A>; fn map2<A, B, C, F>(fa: Self::Wrapped<A>, fb: Self::Wrapped<B>, f: F) -> Self::Wrapped<C>; }
pub trait Traverse: Functor {
    fn traverse<G, A, B, F>(
        fa: Self::Wrapped<A>,
        f: F
    ) -> G::Wrapped<Self::Wrapped<B>>
    where G: Applicative;
}
# fn main() {}
```

## 12.3 Conclusion

Applicative and Traversable allow us to separate the *mechanism* of execution (Sequential vs Parallel) from the *structure* of the data (List vs Tree).
