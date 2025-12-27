# Chapter 12: Applicative and Traversable Functors

In this chapter, we discover that **Monad** is not the only useful abstraction for effectful computation. We introduce **Applicative**, which is generally "weaker" than Monad (it can do less) but also more general (more things are Applicatives) and allows for different behaviors, such as parallel execution and error accumulation. We then generalize the `traverse` and `sequence` functions into the **Traverse** trait.

## 12.1 Applicative Functors

A Monad is defined by `unit` and `flatMap`.
An Applicative is defined by `unit` and `map2` (or `apply`).

```rust
pub trait Applicative: Functor {
    fn unit<A>(a: A) -> Self::Wrapped<A>; // Same as Monad
    
    fn map2<A, B, C, F>(
        fa: Self::Wrapped<A>,
        fb: Self::Wrapped<B>,
        f: F
    ) -> Self::Wrapped<C>;
}
```

### Why Applicative?

If we look at `map2` vs `flatMap`:
- `flatMap` allows the *structure* of the second computation to depend on the *result* of the first.
- `map2` takes two independent computations and combines their results.

This independence means:
1.  **Parallelism**: Since `fa` and `fb` are independent, they can be computed in parallel (e.g., fetching two URLs).
2.  **Analysis**: We can inspect the structure of the computation without running it.
3.  **Accumulation**: We can accumulate errors from both branches (e.g., form validation), whereas `flatMap` usually stops at the first error.

## 12.2 Traversable Functors

We've seen `traverse` and `sequence` for List, Option, and Map. We can abstract this into a **Traverse** trait.

A Traversable functor allows us to iterate over a data structure (like a Tree or List) while maintaining an effect (Applicative) context.

```rust
pub trait Traverse: Functor { // actually extends Functor and Foldable conceptually
    fn traverse<G, A, B, F>(
        fa: Self::Wrapped<A>,
        f: F
    ) -> G::Wrapped<Self::Wrapped<B>>
    where G: Applicative;
}
```

### Example: Tree Traversal

If we have a `Tree<i32>` and we want to apply a function that might fail (returning `Option<i32>`) to every node, `traverse` will give us `Option<Tree<i32>>`. It turns the "Tree of Options" into an "Option of Tree".

```rust
impl Traverse for TreeApp {
    fn traverse<G, A, B, F>(fa: Tree<A>, f: F) -> G::Wrapped<Tree<B>>
    where G: Applicative, ... {
        match fa {
            Tree::Leaf(a) => G::map(f(a), |b| Tree::Leaf(b)),
            Tree::Branch(l, r) => {
                let gl = Self::traverse(*l, f.clone());
                let gr = Self::traverse(*r, f);
                G::map2(gl, gr, |l, r| Tree::Branch(Box::new(l), Box::new(r)))
            }
        }
    }
}
```

## 12.3 Conclusion

Applicative and Traversable complete our tour of the core functional type classes. They allow us to write extremely generic code for data manipulation, separating the *mechanism* of the effect (Option, Result, Async) from the *structure* of the data (List, Tree, Map). In Rust, using generic traits and GATs allows us to express these powerful abstractions in a type-safe way.
