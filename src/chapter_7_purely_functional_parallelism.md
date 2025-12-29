# Chapter 7: Parallel Agent Orchestration (Purely Functional Parallelism)

In this chapter, we will design a purely functional library for creating **Parallel Agent Workflows**. As we did in previous chapters, we won't jump straight to the implementation. Instead, we'll follow a process of *designing the API first*. We'll verify that our API is expressive enough to handle complex Agentic patterns like "Map-Reduce Research" or "Parallel Tool Execution".

This journey is particularly interesting in Rust. The compiler forces us to be rigorous about ownership and sharing across threads from day one. We will see how traits like `Send`, `Sync`, and types like `Arc` become fundamental building blocks of our Agent Orchestrator.

## 7.1 Designing the Agent Orchestrator

Our goal is to create a library that can describe parallel agent tasks. Let's imagine we want to **research a topic** by searching multiple sources in parallel. We might use a divide-and-conquer approach:

```rust
fn gather_research(sources: &[String]) -> String {
    if sources.len() <= 1 {
        let source = sources.get(0).cloned().unwrap_or_default();
        search_source(&source) // Assume this is slow!
    } else {
        let (l, r) = sources.split_at(sources.len() / 2);
        let left_result = gather_research(l);
        let right_result = gather_research(r);
        format!("{}\n{}", left_result, right_result)
    }
}
# fn search_source(s: &str) -> String { format!("Content from {}", s) }
# fn main() {
#     let sources = vec!["Wiki".to_string(), "News".to_string()];
#     let res = gather_research(&sources);
#     assert!(res.contains("Wiki"));
# }
```

This implementation is sequential. To make it parallel, we need a way to say "search `l` and `r` in parallel". Let's invent a container type, let's call it `AgentTask<A>`, that represents a computation of type `A` that *might* be running in another thread (another Agent).

We need a way to take an evaluated result and wrap it (`completed_task`).
And we need a way to get the final result out (`await_result`).

```rust
pub struct AgentTask<A>(std::marker::PhantomData<A>);

impl<A> AgentTask<A> {
    pub fn completed_task(a: A) -> AgentTask<A> { AgentTask(std::marker::PhantomData) }
    pub fn await_result(self) -> A { unimplemented!() }
}
# fn main() {}
```

If we change `gather_research` to use `AgentTask`:

```rust
# struct AgentTask<A>(A);
# impl<A> AgentTask<A> { fn completed_task(a: A) -> AgentTask<A> { AgentTask(a) } fn await_result(self) -> A { self.0 } }
# fn search_source(s: &str) -> String { format!("{}", s) }
fn gather_research(sources: &[String]) -> String {
    if sources.len() <= 1 {
        let source = sources.get(0).cloned().unwrap_or_default();
        search_source(&source)
    } else {
        let (l, r) = sources.split_at(sources.len() / 2);
        // This is still sequential if evaluated immediately!
        let left_task: AgentTask<String> = AgentTask::completed_task(gather_research(l)); 
        let right_task: AgentTask<String> = AgentTask::completed_task(gather_research(r));
        
        format!("{}\n{}", left_task.await_result(), right_task.await_result())
    }
}
# fn main() {}
```

We need a primitive that takes a *lazy* argument (a closure) to run it in the background. Let's call it `spawn_agent`.

```rust
# struct AgentTask<A>(A);
pub fn spawn_agent<A, F>(task: F) -> AgentTask<A> 
where F: FnOnce() -> AgentTask<A> + Send + 'static { task() }
# fn main() {}
```

In Rust, strict evaluation is the default. To prevent immediate execution, we pass a closure (a thunk). `spawn_agent` takes a thunk that returns an `AgentTask<A>`, and executes it on a worker thread.

However, we need to combine results. We need `join_results` (map2):

```rust
# struct AgentTask<A>(A);
pub fn join_results<A, B, C, F>(task_a: AgentTask<A>, task_b: AgentTask<B>, merger: F) -> AgentTask<C>
where F: Fn(A, B) -> C { unimplemented!() }
# fn main() {}
```

With `completed_task`, `spawn_agent`, and `join_results`, our parallel researcher looks like this:

```rust
# struct AgentTask<A>(A);
# impl<A> AgentTask<A> { fn completed_task(a: A) -> Self { AgentTask(a) } fn spawn_agent<F>(f: F) -> Self where F: FnOnce() -> Self { f() } fn join_results<B, C, F>(pa: AgentTask<A>, pb: AgentTask<B>, f: F) -> AgentTask<C> where F: Fn(A, B) -> C { AgentTask(f(pa.0, pb.0)) } }
# fn search_source(s: &str) -> String { format!("{}", s) }
fn gather_research(sources: &[String]) -> AgentTask<String> {
    if sources.len() <= 1 {
        let source = sources.get(0).cloned().unwrap_or_default();
        AgentTask::completed_task(search_source(&source))
    } else {
        let (l, r) = sources.split_at(sources.len() / 2);
        AgentTask::join_results(
            AgentTask::spawn_agent(|| gather_research(l)),
            AgentTask::spawn_agent(|| gather_research(r)),
            |a, b| format!("{}\n{}", a, b)
        )
    }
}
# fn main() {}
```

This looks purely functional! `gather_research` now returns a *description* of a workflow (`AgentTask<String>`), which we can execute later by calling `await_result`.

## 7.2 A Function Representation for `AgentTask`

What should `AgentTask<A>` actually *be*?
It is a function that takes an `Executor` (The Runtime) and returns a `Future[A]` (The Pending Result).

```rust
use std::sync::Arc;
use std::any::Any;

# pub trait Executor {}
# pub trait Future { type Item; }
#[allow(clippy::type_complexity)]
pub struct AgentTask<A>(Arc<dyn Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync>);
# fn main() {}
```

## 7.3 Combinators

Now we can implement the combinators.

### `completed_task` (unit) and `lazy_task`
`completed_task` wraps a value immediately.
`lazy_task` wraps a computation lazily by combining `completed_task` and `spawn_agent`.

```rust
# use std::sync::Arc;
# pub trait Executor {}
# pub trait Future { type Item; }
# pub struct UnitFuture<A>(A);
# impl<A> Future for UnitFuture<A> { type Item = A; }
# pub struct AgentTask<A>(Arc<dyn Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync>);
# impl<A> AgentTask<A> { fn new<F>(f: F) -> Self where F: Fn(&dyn Executor) -> Box<dyn Future<Item=A>> + Send + Sync + 'static { AgentTask(Arc::new(f)) } }
# fn spawn_agent<A, F>(f: F) -> AgentTask<A> where F: Fn() -> AgentTask<A> { unimplemented!() }
pub fn completed_task<A: Clone + Send + Sync + 'static>(a: A) -> AgentTask<A> {
    AgentTask::new(move |_| Box::new(UnitFuture(a.clone())))
}

pub fn lazy_task<A, F>(a: F) -> AgentTask<A> 
where A: Clone + Send + Sync + 'static, F: Fn() -> A + Send + Sync + 'static + Clone {
    spawn_agent(move || completed_task(a()))
}
# fn main() {}
```

### `join_results` (map2)
`join_results` is where the magic happens. It combines two parallel agent tasks.

### `spawn_agent` (fork)
`spawn_agent` is responsible for shifting execution to a separate worker thread.

## 7.4 Laws and Deadlocks

An important property we expect is:
`spawn_agent(x) == x`

Isolating a task in a sub-agent shouldn't change the result, only *where* it runs.

## 7.5 Derived Combinators

Using `join_results`, `completed_task`, and `spawn_agent`, we can derive:

*   `async_function`: Lift a tool `A -> B` to `A -> AgentTask<B>`.
*   `sequence_tasks`: Convert `Vec<AgentTask<A>>` to `AgentTask<Vec<A>>`.
*   `parallel_filter`: Filter a list using a parallel predicate (e.g., "Is this document relevant?").

### Exercise 7.11: `choice` (Router)
We realized we can choose between two tasks based on a boolean condition (e.g. "If tool failed, try fallback").

```rust
# struct AgentTask<A>(A);
pub fn router<A>(cond: AgentTask<bool>, t: AgentTask<A>, f: AgentTask<A>) -> AgentTask<A> {
    unimplemented!()
}
# fn main() {}
```

### Exercise 7.13: `flat_map` (Dynamic Planner)
The most general form of dynamic choice is `flat_map`:

```rust
# struct AgentTask<A>(A);
pub fn dynamic_planner<A, B, F>(task: AgentTask<A>, next_step: F) -> AgentTask<B>
where F: Fn(A) -> AgentTask<B> {
    unimplemented!()
}
# fn main() {}
```

## Summary

In this chapter, we built a functional Agent Orchestrator. We saw how:
1.  **Pure descriptions** (`AgentTask`) separate the "Plan" from the "Runtime".
2.  **Strictness in Rust** requires explicit thunks or closures.
3.  **Thread safety** in Rust ensures our Agents don't have race conditions.

In the next chapter, we will explore **Eval Benchmarks**, where we will use generative testing to verify our Agents behave correctly.
