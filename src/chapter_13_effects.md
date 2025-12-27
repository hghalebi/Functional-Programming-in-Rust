# Chapter 13: External Effects and I/O

This chapter explores how to handle "external effects" (like reading files, printing to console, or network calls) in a purely functional way. The core idea is to separate the **description** of the computation from its **execution**.

> [!NOTE]
> In idiomatic Rust, I/O is typically handled imperatively with `std::io` or asynchronously with `Future` (which is monad-like!). This chapter explores the *pure functional modeling* of I/O, which powers the design of async runtimes.

## 13.1 The IO Type

We can model an effectful computation as a value that, when "run", performs the effect. In its simplest form, this is just a wrapper around a function (a thunk).

```rust
pub struct SimpleIO<A>(Box<dyn FnOnce() -> A>);

impl<A> SimpleIO<A> {
    pub fn new<F>(f: F) -> Self where F: FnOnce() -> A + 'static { 
        SimpleIO(Box::new(f)) 
    }
    
    pub fn run(self) -> A { (self.0)() }
    
    pub fn flat_map<B, F>(self, f: F) -> SimpleIO<B>
    where F: FnOnce(A) -> SimpleIO<B> + 'static
    {
        SimpleIO::new(move || f(self.run()).run())
    }
}
```

## 13.2 Composition

Since `IO` is a description, we can compose these descriptions. `IO` forms a Monad.

- **`map`**: Transform the result of an effect description.
- **`flatMap`**: Chain effects, where the second effect depends on the result of the first.

```rust
let program = Console::read_line()
    .flatMap(|name| Console::print_line(&format!("Hello, {}!", name)));
```

Until we call `program.run()`, nothing happens. This is **Referential Transparency**: the expression `program` can be replaced by its definition without changing the outcome (because the outcome *is just a description*, not the side effect itself).

## 13.3 Limitations and The Free Monad

Our simple `IO` type has a flaw: it uses the Rust call stack for `flatMap`. Deeply recursive programs (like an infinite event loop) will cause a `StackOverflowError`.

In Scala (and advanced Rust), we solve this using **Trampolining** or the **Free Monad**. This involves reifying the control flow (`FlatMap`, `Return`) as data structures rather than function calls. The interpreter then runs in a loop (a trampoline), keeping the stack constant.

Implementing a full generic `Free` monad in Rust is complex due to the lack of Higher-Kinded Types (HKT) and challenges with type erasure in `FlatMap`. Ideally, one would use a crate like `stacker` or specialized enum structures, but the conceptual model remains the same: **Data as Control Flow**.

## 13.4 Use Cases

This pattern allows us to:
1.  **Test Effectful Code**: We can inspect the description (if we use the `Free` enum approach) or swap interpreters.
2.  **Safe Refactoring**: We can refactor effectful sequences with the same confidence as pure logic.
3.  **Custom Interpreters**: We could run an `IO` program asynchronously, or log every step, just by changing the `run` function.

> [!TIP]
> Rust's `async`/`await` syntax effectively builds a state machine (a form of Free Monad or Coroutine) that the Executor (like `tokio`) interprets. Understanding `IO` monads clarifies how `async` works under the hood!
