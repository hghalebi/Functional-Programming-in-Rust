# Chapter 5: Token Streaming (Strictness and Laziness)

In this chapter, we explore **non-strictness** (laziness) to improve efficiency and modularity. In the context of LLMs, this is crucial for **Token Streaming**. We don't want to wait for the entire 4,000-token response to be generated before showing the first word. We need a `TokenStream` that fuses sequences of transformations (like filtering or stopping criteria) without realizing the full buffer.

## 5.1 Strict and non-strict functions

Strict functions evaluate their arguments *before* the function body is executed. Non-strict functions may choose not to evaluate arguments.

In Rust, arguments are strictly evaluated by default. To simulate non-strictness (lazy loading), we can pass a closure (thunk) `Fn() -> A` instead of a value `A`.

```rust
// logical AND is non-strict in the second argument
// stop_sequence_met && { println!("Generation stopped!"); true } 
```

## 5.2 An extended example: Token Streams

We define a `TokenStream` (lazy list). In Rust, we use closures wrapped in `Rc` (for sharing) to represent these thunks (un-generated tokens).

```rust
use std::rc::Rc;

#[derive(Clone)]
pub enum TokenStream<A> {
    Empty,
    Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> TokenStream<A>>),
}
// Note: We avoid memoization complexity for this basic translation. 
// In a production specific persistent stream, one might use `lazy_static` or `OnceCell`.
# fn main() {
#    // verify basic construction
#    let _ = TokenStream::Cons(Rc::new(|| "Hello"), Rc::new(|| TokenStream::Empty));
# }
```

### Exercise 5.1: collect_text (to_vec)
Force the stream into a strict `Vec` (Full Response).

```rust
# use std::rc::Rc;
# #[derive(Clone)] pub enum TokenStream<A> { Empty, Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> TokenStream<A>>) }
// impl TokenStream<A>
pub fn collect_text<A>(s: &TokenStream<A>) -> Vec<A> { 
    match s {
        TokenStream::Empty => Vec::new(),
        TokenStream::Cons(h, t) => {
            let mut v = vec![h()];
            v.extend(collect_text(&t()));
            v
        }
    }
}
# fn main() {}
```

### Exercise 5.2: limit_tokens (take) and skip_preamble (drop)
```rust
# use std::rc::Rc;
# #[derive(Clone)] pub enum TokenStream<A> { Empty, Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> TokenStream<A>>) }
# impl<A> TokenStream<A> { fn cons<F, S>(h: F, t: S) -> Self where F: Fn() -> A + 'static, S: Fn() -> TokenStream<A> + 'static { TokenStream::Cons(Rc::new(h), Rc::new(t)) } }
pub fn limit_tokens<A: 'static>(s: &TokenStream<A>, n: usize) -> TokenStream<A> {
    if n == 0 {
        TokenStream::Empty
    } else {
        match s {
            TokenStream::Empty => TokenStream::Empty,
            TokenStream::Cons(h, t) => {
                let h = h.clone();
                let t = t.clone();
                TokenStream::cons(move || h(), move || limit_tokens(&t(), n - 1))
            }
        }
    }
}
# fn main() {}
```

### Exercise 5.3: take_while
Return all tokens until a Stop Sequence is met.

## 5.3 Separating program description from evaluation

Laziness lets us separate the description of an expression (the pipeline) from its evaluation (the generation).

### Exercise 5.4: validate_stream (for_all)
Check that all tokens in the Stream match a given predicate (e.g. "Is Safe"), terminating early if a violation occurs.

### Exercise 5.5: take_while using fold_right
### Exercise 5.6: head_option (peek_first_token)
### Exercise 5.7: map, filter, append, flat_map using fold_right

## 5.4 Infinite streams (Heartbeats)

Because functions are incremental, they work for infinite streams (e.g., a "Waiting" animation or Heartbeat signal).

```rust
# use std::rc::Rc;
# #[derive(Clone)] pub enum TokenStream<A> { Empty, Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> TokenStream<A>>) }
# impl<A> TokenStream<A> { fn cons<F, S>(h: F, t: S) -> Self where F: Fn() -> A + 'static, S: Fn() -> TokenStream<A> + 'static { TokenStream::Cons(Rc::new(h), Rc::new(t)) } }
pub fn heartbeats() -> TokenStream<&'static str> {
    TokenStream::cons(|| ".", heartbeats)
}
# fn main() {}
```

### Exercise 5.8: constant
### Exercise 5.9: from
### Exercise 5.10: fibs (loading_simulation)
### Exercise 5.11: unfold (generate_tokens)
A general stream-building function (corecursion). Ideal for wrapping an LLM API that returns chunks.

```rust
# use std::rc::Rc;
# #[derive(Clone)] pub enum TokenStream<A> { Empty, Cons(Rc<dyn Fn() -> A>, Rc<dyn Fn() -> TokenStream<A>>) }
# impl<A> TokenStream<A> { fn cons<F, S>(h: F, t: S) -> Self where F: Fn() -> A + 'static, S: Fn() -> TokenStream<A> + 'static { TokenStream::Cons(Rc::new(h), Rc::new(t)) } }
pub fn generate_tokens<A, S, F>(initial_state: S, generator: F) -> TokenStream<A>
where A: Clone + 'static, S: Clone + 'static, F: Fn(S) -> Option<(A, S)> + 'static + Clone {
    match generator(initial_state) {
        Some((token, next_state)) => {
            let gen = generator.clone();
            TokenStream::cons(move || token.clone(), move || generate_tokens(next_state.clone(), gen.clone()))
        },
        None => TokenStream::Empty,
    }
}
# fn main() {}
```

### Exercise 5.12: fibs, from, constant, via unfold
### Exercise 5.13: map, take, take_while, zip_with, zip_all via unfold
### Exercise 5.14: starts_with
### Exercise 5.15: tails
### Exercise 5.16: scan_right

## 5.5 Summary
Laziness improves modularity by decoupling the description of a token pipeline from its execution.

## 5.6 References

*   **Rust Book Ch 13 (Iterators)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch13-02-iterators.html)
*   **Rust By Example (Iterators)**: [Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html)
*   **Rust Patterns (Lazy Evaluation)**: [Idioms](https://rust-unofficial.github.io/patterns/idioms/lazy-evaluation.html)
