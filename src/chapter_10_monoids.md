# Chapter 10: Map-Reduce Summarization (Monoids)

In this chapter, we explore one of the simplest and most ubiquitous algebraic structures: the **Summarizer** (Monoid). This serves as our introduction to purely algebraic structures.

In Agent systems, we often need to **aggregate** data from multiple sources:
- Merging conversation histories.
- Summing total token usage across parallel agents.
- Coalescing partial JSON objects.

## 10.1 What is a Summarizer?

A Monoid (Summarizer) consists of:
1.  A type `A`.
2.  An associative binary operation `combine(a1, a2): A`. (Merging 2 histories).
3.  An identity element `empty: A`. (Empty history).

```rust
pub trait Summarizer<A> {
    fn combine(&self, a1: A, a2: A) -> A;
    fn empty(&self) -> A;
}
# fn main() {}
```

### Examples

**String Concatenation (History Merge)**:
```rust
# pub trait Summarizer<A> { fn combine(&self, a1: A, a2: A) -> A; fn empty(&self) -> A; }
pub struct HistoryMerger;
impl Summarizer<String> for HistoryMerger {
    fn combine(&self, a1: String, a2: String) -> String { a1 + &a2 }
    fn empty(&self) -> String { "".to_string() }
}
# fn main() {}
```

**Token Usage (Int Addition)**:
```rust
# pub trait Summarizer<A> { fn combine(&self, a1: A, a2: A) -> A; fn empty(&self) -> A; }
pub struct TokenTotal;
impl Summarizer<i32> for TokenTotal {
    fn combine(&self, a1: i32, a2: i32) -> i32 { a1 + a2 }
    fn empty(&self) -> i32 { 0 }
}
# fn main() {}
```

## 10.2 Parallel Summarization (Map-Reduce)

If we have a huge context window `[Message1, Message2, ...]`, we can measure its token cost in parallel by splitting it.

```rust
# pub trait Summarizer<A> { fn combine(&self, a1: A, a2: A) -> A; fn empty(&self) -> A; }
pub fn parallel_summarize<A, B, M, F>(v: &[A], m: &M, f: &F) -> B
where M: Summarizer<B>, F: Fn(&A) -> B + Clone, B: Clone {
    if v.is_empty() {
        m.empty()
    } else if v.len() == 1 {
        f(&v[0])
    } else {
        let (left, right) = v.split_at(v.len() / 2);
        let lb = parallel_summarize(left, m, f);
        let rb = parallel_summarize(right, m, f);
        m.combine(lb, rb)
    }
}
# fn main() {}
```

## 10.4 Example: Token Counting (Word Count)

A practical use case is counting tokens in a large corpus without encoding everything sequentially.

```rust
pub enum TokenStats {
    Fragment(String), // Partial token
    Count(String, i32, String), // (Left partial, count of full tokens, Right partial)
}
# fn main() {}
```

## 10.5 Conclusion

Summarizers (Monoids) provide a simple abstraction for parallel aggregation. This sets the stage for more complex algebraic structures like **Generic Agents** (Functors and Monads), which we will explore in the next chapters.
