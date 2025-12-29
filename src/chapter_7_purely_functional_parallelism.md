# Chapter 7: Purely Functional Parallelism

In this chapter, we will explore the design of a purely functional library for creating parallel and asynchronous computations. As we did in previous chapters, we won't jump straight to the implementation. Instead, we'll follow a process of *designing the API first*. We'll verify that our API is expressive and follows algebraic laws, and only *then* will we worry about how to implement it efficiently.

This journey is particularly interesting in Rust compared to Scala. In Scala, we can often be loose with thread safety due to the garbage collector and the JVM's memory model. In Rust, however, the compiler forces us to be rigorous about ownership and sharing across threads from day one. We will see how traits like `Send`, `Sync`, and types like `Arc` become fundamental building blocks of our parallel algebra.

## 7.1 Choosing a Data Type and Functions

Our goal is to create a library that can describe parallel computations. Let's imagine we want to sum a list of integers in parallel. We might use a divide-and-conquer approach:

```rust
fn sum(ints: &[i32]) -> i32 {
    if ints.len() <= 1 {
        *ints.get(0).unwrap_or(&0) // equivalent to headOption.getOrElse(0)
    } else {
        let (l, r) = ints.split_at(ints.len() / 2);
        let sum_l = sum(l);
        let sum_r = sum(r);
        sum_l + sum_r
    }
}
# fn main() {
#     assert_eq!(sum(&[1, 2, 3, 4]), 10);
# }
```

This implementation is sequential. To make it parallel, we need a way to say "compute `sum_l` and `sum_r` in parallel". Let's invent a container type, let's call it `Par<A>` (short for Parallel), that represents a computation of type `A` that *might* be running in another thread.

We need a way to take an unevaluated `A` and wrap it in a `Par<A>`. In Scala, this is `Par.unit`.
And we need a way to get the result out. Scala calls this `run`.

```rust
pub struct Par<A>(std::marker::PhantomData<A>);

impl<A> Par<A> {
    pub fn unit(a: A) -> Par<A> { Par(std::marker::PhantomData) }
    pub fn run(self) -> A { unimplemented!() }
}
# fn main() {}
```

If we change `sum` to use `Par`:

```rust
# struct Par<A>(A);
# impl<A> Par<A> { fn unit(a: A) -> Par<A> { Par(a) } fn run(self) -> A { self.0 } }
fn sum(ints: &[i32]) -> i32 {
    if ints.len() <= 1 {
        *ints.get(0).unwrap_or(&0)
    } else {
        let (l, r) = ints.split_at(ints.len() / 2);
        // unit returns Par<i32> immediately
        let sum_l: Par<i32> = Par::unit(sum(l)); 
        let sum_r: Par<i32> = Par::unit(sum(r));
        // get results (run)
        sum_l.run() + sum_r.run()
    }
}
# fn main() {}
```

But wait! `Par::unit` takes an *evaluated* value `A`. If we pass `sum(l)` to it, `sum(l)` is evaluated *before* `unit` is called, on the current thread. We haven't achieved parallelism; we've just wrapped the result!

We need a primitive that takes a *lazy* argument or a closure. Let's call it `fork`.

```rust
# struct Par<A>(A);
pub fn fork<A, F>(a: F) -> Par<A> 
where F: FnOnce() -> Par<A> + Send + 'static { a() }
# fn main() {}
```

In Rust, strict evaluation is the default. To prevent immediate execution, we pass a closure (a thunk). `fork` should take a thunk that returns a `Par<A>`, and run that thunk in a separate thread.

However, `fork` shouldn't just run *anything*. It specifically handles `Par`. To combine results, we need `map2`:

```rust
# struct Par<A>(A);
pub fn map2<A, B, C, F>(pa: Par<A>, pb: Par<B>, f: F) -> Par<C>
where F: Fn(A, B) -> C { unimplemented!() }
# fn main() {}
```

With `unit`, `fork`, and `map2`, our `sum` looks like this:

```rust
# struct Par<A>(A);
# impl<A> Par<A> { fn unit(a: A) -> Self { Par(a) } fn fork<F>(f: F) -> Self where F: FnOnce() -> Self { f() } fn map2<B, C, F>(pa: Par<A>, pb: Par<B>, f: F) -> Par<C> where F: Fn(A, B) -> C { Par(f(pa.0, pb.0)) } }
fn sum(ints: &[i32]) -> Par<i32> {
    if ints.len() <= 1 {
        Par::unit(*ints.get(0).unwrap_or(&0))
    } else {
        let (l, r) = ints.split_at(ints.len() / 2);
        Par::map2(
            Par::fork(|| sum(l)),
            Par::fork(|| sum(r)),
            |a, b| a + b
        )
    }
}
# fn main() {}
```

This looks purely functional! `sum` now returns a *description* of a parallel computation (`Par<i32>`), which we can execute later by calling `run`.

## 7.2 A Function Representation for `Par`

What should `Par<A>` actually *be*?
In the book, `Par[A]` is defined as a function that takes an `ExecutorService` and returns a `Future[A]`.

```scala
type Par[A] = ExecutorService => Future[A]
```

In Rust, this translates to something slightly more complex due to ownership.
We need an `Executor` trait (similar to `ExecutorService`).
We need a `Future` trait.
And `Par<A>` is a closure.

### The Rust Implementation Challenges

1.  **Object Safety**: We want to pass `&dyn Executor` around. This requires `Executor` to be "object safe". We cannot have generic methods like `submit<T>(...)` in an object-safe trait. We must use type erasure, e.g., `Box<dyn Any + Send>`.
2.  **Thread Safety**: Since `Par` is executed potentially on other threads, the closure defining it must be `Send` and `Sync`. We use `Arc` instead of `Rc` to allow shared ownership across threads.

Here is our refined definition:

```rust
use std::sync::Arc;
use std::any::Any;

# pub trait Executor {}
# pub trait Future { type Item; }
#[allow(clippy::type_complexity)]
pub struct Par<A>(Arc<dyn Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync>);
# fn main() {}
```

This says: `Par<A>` assumes it can be shared (`Arc`), run concurrently (`Sync`), and sent across threads (`Send`). It takes an `Executor` and produces a `Future` yielding `A`.

## 7.3 Combinators

Now we can implement the combinators.

### `unit` and `lazy_unit`

`unit` wraps a value immediately.
`lazy_unit` wraps a computation lazily by combining `unit` and `fork`.

```rust
# use std::sync::Arc;
# pub trait Executor {}
# pub trait Future { type Item; }
# pub struct UnitFuture<A>(A);
# impl<A> Future for UnitFuture<A> { type Item = A; }
# pub struct Par<A>(Arc<dyn Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync>);
# impl<A> Par<A> { fn new<F>(f: F) -> Self where F: Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync + 'static { Par(Arc::new(f)) } }
# fn fork<A, F>(f: F) -> Par<A> where F: Fn() -> Par<A> { unimplemented!() }
pub fn unit<A: Clone + Send + Sync + 'static>(a: A) -> Par<A> {
    Par::new(move |_| Box::new(UnitFuture(a.clone())))
}

pub fn lazy_unit<A, F>(a: F) -> Par<A> 
where A: Clone + Send + Sync + 'static, F: Fn() -> A + Send + Sync + 'static + Clone {
    fork(move || unit(a()))
}
# fn main() {}
```

> **Note**: The `Clone` bounds are often necessary in our simple implementation because the description of a parallel computation might be reused or re-executed. In a production library like `rayon`, generic lifetimes are handled more carefully to avoid excessive cloning.

### `map2`

`map2` is where the magic happens. It combines two parallel computations.

```rust
# struct Par<A>(Box<dyn Fn()>, std::marker::PhantomData<A>);
# impl<A> Par<A> { fn new<F>(_: F) -> Self { Par(Box::new(||()), std::marker::PhantomData) } }
# struct UnitFuture<A>(A);
pub fn map2<A, B, C, F>(pa: Par<A>, pb: Par<B>, f: F) -> Par<C> 
where F: Fn(A, B) -> C + 'static {
    Par::<C>::new(move |_: &()| {
        // Mock impl
        unimplemented!() 
    })
}
# fn main() {}
```

### `fork`

`fork` is responsible for shifting execution to a separate logical thread.

```rust,ignore
pub fn fork<A, F>(a: F) -> Par<A> {
    Par::new(move |es| {
        // Submit a task to the executor
        let future = es.submit(wrapped_task);
        // Return a future that waits for the result
        ...
    })
}
```

## 7.4 Laws and Deadlocks

An important property we expect is:
`fork(x) == x`

Forking a computation shouldn't change its result, only *where* it runs.
However, with a fixed-size thread pool, naïve implementations of `fork` can deadlock!
If the thread pool has 1 thread:
1. `fork(fork(x))` is called.
2. The outer `fork` consumes the thread.
3. It tries to run the inner `fork`.
4. The inner `fork` waits for a thread... but the only thread is blocked waiting for the inner `fork`!

This teaches us that **API design is not just about types, but about runtime semantics and laws**. A correct `Par` implementation for fixed-size pools requires non-blocking futures (using callbacks), which allows the thread to be released while waiting.

## 7.5 Derived Combinators

Using `map2`, `unit`, and `fork`, we can derive:

*   `async_f`: Lift a function `A -> B` to `A -> Par<B>`.
*   `sequence`: Convert `Vec<Par<A>>` to `Par<Vec<A>>`.
*   `par_filter`: Filter a list in parallel.
*   `map`: Derive `map` from `map2` and `unit`.

### Exercise 7.11: `choice`

We realized we can choose between two computations based on a boolean condition:

```rust
# struct Par<A>(A);
pub fn choice<A>(cond: Par<bool>, t: Par<A>, f: Par<A>) -> Par<A> {
    unimplemented!()
}
# fn main() {}
```

This naturally generalizes to `choice_n` (choosing from a list) and finally:

### Exercise 7.13: `flat_map` (chooser)

The most general form of dynamic choice is `flat_map`:

```rust
# struct Par<A>(A);
pub fn flat_map<A, B, F>(pa: Par<A>, f: F) -> Par<B>
where F: Fn(A) -> Par<B> {
    unimplemented!()
}
# fn main() {}
```

`flat_map` allows the structure of the computation to depend on the *result* of previous computations.

## Summary

In this chapter, we built a functional parallelism library. We saw how:
1.  **Pure descriptions** (`Par`) separate the "what" from the "how" (the `Executor`).
2.  **Strictness in Rust** requires explicit thunks or closures (like `F: Fn() -> A`) to achieve laziness.
3.  **Thread safety** in Rust is explicit (`Send`, `Sync`, `Arc`), ensuring our parallel library prevents data races at compile time—a guarantee Scala cannot easily provide.

In the next chapter, we will explore **Property-Based Testing**, where we will use generative testing to verifying laws like `fork(x) == x` automatically.
