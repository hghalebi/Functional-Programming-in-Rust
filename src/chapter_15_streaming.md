# Chapter 15: Stream Processing and Incremental I/O

The final step in our journey is to handle I/O incrementally. We want to process data (like lines in a large file) without loading everything into memory (like `List` or `Vec`), but still retain the composability of `map`, `filter`, and `fold`.

## 15.1 The Problem with I/O

Standard imperative I/O mixes the *what* (logic) with the *how* (looping, reading). 
Standard functional `List` separates them but requires loading all data into memory first.

## 15.2 The Process Algebra

We define `Process<I, O>` as a state machine that can:
1.  **Emit** a value of type `O` and transition to a new state.
2.  **Await** a value of type `I` and transition to a new state.
3.  **Halt**.

```rust
pub enum Process<I, O> {
    Emit(O, Box<Process<I, O>>),
    Await(Box<dyn FnOnce(Option<I>) -> Process<I, O>>),
    Halt,
}
# fn main() {}
```

This is a **Pull-based** stream. The driver calls the process; if the process is `Emit`ting, the driver collects the value. If the process is `Await`ing, the driver fetches inputs (e.g., from a file) and feeds them in.

## 15.3 Composition: The Pipe (`|>`)

The true power comes from `pipe`. We can feed the output of one process into the input of another.

```rust
# enum Process<I, O> { Emit(O, Box<Process<I, O>>), Await(Box<dyn FnOnce(Option<I>) -> Process<I, O>>), Halt }
# impl<I, O> Process<I, O> {
#     fn filter<F>(_: F) -> Process<I, I> where F: Fn(&I) -> bool { Process::Halt }
#     fn lift<F>(_: F) -> Process<I, O> where F: Fn(I) -> O { Process::Halt }
#     fn pipe<O2>(self, _: Process<O, O2>) -> Process<I, O2> { Process::Halt }
# }
let p1 = Process::filter(|x| x % 2 == 0); // Producers/Transducers
let p2 = Process::lift(|x: i32| x * 10);
let pipeline = p1.pipe(p2); // Fused Process
# fn main() {}
```

The `pipe` implementation fuses the two machines into one. It runs `p2` until `p2` awaits input, then run `p1` to produce that input. This ensures **constant memory usage** (processing one element at a time) for arbitrary pipelines.

## 15.4 Conclusion

This architecture (often called **Iteratees**, **Transducers**, or **Pull Streams**) is the foundation of modern functional streaming libraries like `fs2` (Scala) or `futures::Stream` (Rust). It allows us to process infinite streams or massive files with the same elegance as operating on small lists.
