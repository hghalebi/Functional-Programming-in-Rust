# Chapter 14: Scratchpads (Local Effects)

In Agentic programming, we maintain purity in our interfaces ("Input -> Output"). However, some internal reasoning steps (like sorting evidence, accumulating stats, or graph search) are naturally expressed with mutable state. The types `Scratchpad` (ST) and `ScratchRef` allow an Agent to use mutable memory *locally* while remaining externally pure.

## 14.1 Purely Functional In-Place Reasoning?

It sounds like a contradiction. How can we have mutation in a pure function?
The key is **Scope**.
If an Agent allocates a mutable "Scratchpad", scribbles on it, and then returns a frozen summary, and *no one else* ever saw the scratchpad, then the Agent appears pure from the outside.

## 14.2 The Scratchpad Monad

We implemented `Scratchpad<'a, A>` where `'a` represents the "thread" or scope of the scratchpad.

```rust
pub struct Scratchpad<'a, A> {
    run: Box<dyn FnOnce() -> A + 'a>, // Closure capturing mutable 'a references
}
# fn main() {}
```

### Preventing Leakage (Data Privacy)

The danger is leaking a mutable reference to the global scope (Hallucinating internal state into the final answer).
Rust's Higher-Rank Trait Bounds (HRTB) prevent this leakage. The phantom lifetime `'a` ensures that `ScratchRef` cannot escape the `run_scratchpad` block.

```rust
# struct ScratchRef<'a, T>(std::marker::PhantomData<&'a T>);
# impl<'a, T> ScratchRef<'a, T> { fn new(_: T) -> Self { ScratchRef(std::marker::PhantomData) } }
# struct Scratchpad<'a, A>(std::marker::PhantomData<&'a A>);
# fn run_scratchpad<A, T>(_: ScratchRef<'_, T>) { }
// let leaking_ref = run_scratchpad(ScratchRef::new(1)); // ERROR! Result cannot be ScratchRef
# fn main() {}
```

## 14.3 Comparison with Rust References

In a sense, **Rust is the Scratchpad Monad**. Every block of code with local variables is an implicit `Scratchpad` computation. However, the explicit `Scratchpad` monad is useful when we want to treat "stateful reasoning" as First-Class Values.

## 14.4 Conclusion

We now have the ability to write efficient, in-place algorithms (like "Evidence Sorting") using the `Scratchpad` monad, wrapping them safely so they appear pure to the rest of the orchestration system.
