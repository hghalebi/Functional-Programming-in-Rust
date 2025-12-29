# Chapter 15: Streaming Pipelines

The final step in our journey is to handle I/O incrementally. We want to process data (like streams of tokens from an LLM) without loading everything into memory, but still retain the composability of `map`, `filter`, and `fold`.

## 15.1 The Problem with LLM Streams

Standard imperative loops mix the *logic* (what to detect) with the *mechanics* (how to buffer tokens).
Standard functional `List` requires waiting for the full response (high latency).

## 15.2 The Pipeline Algebra

We define `Pipeline<I, O>` as a state machine that can:
1.  **YieldEvent** a value of type `O`.
2.  **AwaitInput** of type `I`.
3.  **Halt**.

```rust
pub enum Pipeline<I, O> {
    YieldEvent(O, Box<Pipeline<I, O>>),
    AwaitInput(Box<dyn FnOnce(Option<I>) -> Pipeline<I, O>>),
    Halt,
}
# fn main() {}
```

This is a **Pull-based** stream. The driver calls the pipeline; if it yields, we show the token. If it awaits, we fetch the next token from the API.

## 15.3 Composition: The Pipe (`|>`)

The true power comes from `pipe`. We can feed the output of one pipeline (e.g. `Tokenizer`) into the input of another (e.g. `StopSequenceDetector`).

```rust
# enum Pipeline<I, O> { YieldEvent(O, Box<Pipeline<I, O>>), AwaitInput(Box<dyn FnOnce(Option<I>) -> Pipeline<I, O>>), Halt }
# impl<I, O> Pipeline<I, O> {
#     fn filter<F>(_: F) -> Pipeline<I, I> where F: Fn(&I) -> bool { Pipeline::Halt }
#     fn lift<F>(_: F) -> Pipeline<I, O> where F: Fn(I) -> O { Pipeline::Halt }
#     fn pipe<O2>(self, _: Pipeline<O, O2>) -> Pipeline<I, O2> { Pipeline::Halt }
# }
let p1 = Pipeline::<i32, i32>::filter(|token_id| *token_id != 0); // Filter padding
let p2 = Pipeline::<i32, i32>::lift(|x| x); // Pass through
let pipeline = p1.pipe(p2); // Fused Pipeline
# fn main() {}
```

The `pipe` implementation fuses the two machines into one. It ensures **constant memory usage** (processing one token at a time).

## 15.4 Conclusion

This architecture is the foundation of modern Agentic streaming libraries. It allows us to process infinite streams of reasoning or massive context contexts with elegance.
