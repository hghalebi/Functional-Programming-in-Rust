# Chapter 4: Handling Hallucinations (Error Handling)

Throwing exceptions is a side effect. In Agentic programming, we often deal with unreliable components: LLMs hallucinate, Tools fail, and APIs timeout. We prefer representing these failures as **ordinary values**. This preserves referential transparency and allows our Agents to reason about their own failures.

In this chapter, we re-create two standard library types: `Option` (Missing values) and `Either` (Failures with reasons).

## 4.1 The Option data type

We represent the possibility of an undefined result (e.g., "Tool produced no output" or "Regex failed to capture") with `Option`.

```rust
pub enum Option<A> {
    Some(A),
    None,
}
# fn main() {}
```

### Exercises 4.1: Basic Functions

Implement `map`, `flat_map`, `get_or_else`, `or_else`, and `filter`.

```rust
# #[derive(Clone, Copy)] pub enum Option<A> { Some(A), None }
impl<A> Option<A> {
    pub fn map<B, F>(self, f: F) -> Option<B> 
    where F: FnOnce(A) -> B {
        match self {
            Option::Some(a) => Option::Some(f(a)),
            Option::None => Option::None,
        }
    }

    pub fn flat_map<B, F>(self, f: F) -> Option<B>
    where F: FnOnce(A) -> Option<B> {
        match self {
            Option::Some(a) => f(a),
            Option::None => Option::None, // Failure propagates
        }
    }

    pub fn get_or_else(self, default: A) -> A {
        match self {
            Option::Some(a) => a,
            Option::None => default, // Fallback value
        }
    }

    pub fn or_else<F>(self, ob: F) -> Option<A>
    where F: FnOnce() -> Option<A> {
        match self {
            Option::Some(_) => self,
            Option::None => ob(), // Try another strategy (Fallback Agent)
        }
    }

    pub fn filter<F>(self, f: F) -> Option<A>
    where F: FnOnce(&A) -> bool {
        match self {
            Option::Some(a) if f(&a) => Option::Some(a),
            _ => Option::None,
        }
    }
}
# fn main() {}
```

### Exercise 4.2: Variance (Latency Analysis)
Implement `variance` for analyzing API latency stability.

```rust
# #[derive(Clone, Copy)] pub enum Option<A> { Some(A), None }
# impl<A> Option<A> { 
#    pub fn flat_map<B, F>(self, f: F) -> Option<B> where F: FnOnce(A) -> Option<B> { match self { Option::Some(a) => f(a), Option::None => Option::None } } 
# }
fn average_latency(latencies: &[f64]) -> Option<f64> {
    if latencies.is_empty() {
        Option::None
    } else {
        Option::Some(latencies.iter().sum::<f64>() / latencies.len() as f64)
    }
}

pub fn latency_variance(latencies: &[f64]) -> Option<f64> {
    average_latency(latencies).flat_map(|m| average_latency(&latencies.iter().map(|x| (x - m).powi(2)).collect::<Vec<_>>()))
}
# fn main() {}
```

### Exercise 4.3: Map2 (Combine Tool Outputs)
Combine two Option values (e.g. `search_result` and `weather_data`). If either failed, the combination fails.

```rust
# #[derive(Clone, Copy)] pub enum Option<A> { Some(A), None }
# impl<A> Option<A> { 
#    pub fn flat_map<B, F>(self, f: F) -> Option<B> where F: FnOnce(A) -> Option<B> { match self { Option::Some(a) => f(a), Option::None => Option::None } } 
#    pub fn map<B, F>(self, f: F) -> Option<B> where F: FnOnce(A) -> B { match self { Option::Some(a) => Option::Some(f(a)), Option::None => Option::None } }
# }
pub fn combine_tool_outputs<A, B, C, F>(a: Option<A>, b: Option<B>, f: F) -> Option<C>
where F: FnOnce(A, B) -> C {
    a.flat_map(|aa| b.map(|bb| f(aa, bb)))
}
# fn main() {}
```

### Exercise 4.4: Sequence (Batch Execution)
Combine a list of optional results into one Option containing a list. If *any* tool failed, the whole batch fails.

```rust
# pub enum Option<A> { Some(A), None }
pub fn sequence_results<A>(a: Vec<Option<A>>) -> Option<Vec<A>> {
    let mut res = Vec::new();
    for opt in a {
        match opt {
            Option::Some(val) => res.push(val),
            Option::None => return Option::None,
        }
    }
    Option::Some(res)
}
# fn main() {}
```

### Exercise 4.5: Traverse (Parallel Execution)
Map a function (e.g., `execute_tool`) over a list of inputs.

```rust
# pub enum Option<A> { Some(A), None }
pub fn traverse_executions<A, B, F>(inputs: Vec<A>, f: F) -> Option<Vec<B>>
where F: Fn(A) -> Option<B> {
    let mut res = Vec::new();
    for x in inputs {
        match f(x) {
            Option::Some(y) => res.push(y),
            Option::None => return Option::None,
        }
    }
    Option::Some(res)
}
# fn main() {}
```

## 4.4 The Either data type

`Option` doesn't tell us *why* something failed (e.g., "Rate Limit Exceeded" vs "Invalid JSON"). `Either` lets us track a failure reason.

```rust
pub enum Either<E, A> {
    Left(E), // The "Error" or "Refusal"
    Right(A), // The "Success" or "Tool Output"
}
# fn main() {}
```

### Exercise 4.6: Basic Functions on Either

```rust
# pub enum Either<E, A> { Left(E), Right(A) }
impl<E, A> Either<E, A> {
    pub fn map<B, F>(self, f: F) -> Either<E, B>
    where F: FnOnce(A) -> B {
        match self {
            Either::Right(a) => Either::Right(f(a)),
            Either::Left(e) => Either::Left(e),
        }
    }
    
    pub fn flat_map<EE, B, F>(self, f: F) -> Either<EE, B>
    where 
        F: FnOnce(A) -> Either<EE, B>,
        EE: From<E> 
    {
        match self {
            Either::Right(a) => f(a),
            Either::Left(e) => Either::Left(EE::from(e)),
        }
    }
    
    pub fn or_else<EE, F>(self, b: F) -> Either<EE, A>
    where 
        F: FnOnce() -> Either<EE, A>,
        EE: From<E>
    {
        match self {
            Either::Right(a) => Either::Right(a),
            Either::Left(_) => b(), // Retry!
        }
    }
    
    pub fn map2<EE, B, C, F>(self, b: Either<EE, B>, f: F) -> Either<EE, C>
    where 
        F: FnOnce(A, B) -> C,
        EE: From<E> 
    {
        self.flat_map(|aa| b.map(|bb| f(aa, bb)))
    }
}
# fn main() {}
```
*Note: In Rust, handling different error types `E` and `EE` usually requires `From` or a common Error trait.*

### Exercise 4.7: Sequence and Traverse for Either

```rust
# pub enum Either<E, A> { Left(E), Right(A) }
pub fn sequence_execution<E, A>(es: Vec<Either<E, A>>) -> Either<E, Vec<A>> {
    traverse_execution(es, |x| x)
}

pub fn traverse_execution<E, A, B, F>(as_vec: Vec<A>, f: F) -> Either<E, Vec<B>>
where F: Fn(A) -> Either<E, B> {
    let mut res = Vec::new();
    for a in as_vec {
        match f(a) {
            Either::Right(b) => res.push(b),
            Either::Left(e) => return Either::Left(e),
        }
    }
    Either::Right(res)
}
# fn main() {}
```

### Exercise 4.8: Accumulating Errors (Validation)
To report *both* "Model Refusal" AND "Context Limit Exceeded", we would need a data structure that can hold multiple errors, like `Validation<Vec<E>, A>`. `Either` stops at the first error (fail-fast).

## 4.5 Summary
We learned to handle Agent failures and hallucinations as values using `Option` and `Either`.

## 4.6 References

*   **Error Handling**: [The Rust Book Ch 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
*   **Recoverable Errors (Result)**: [The Rust Book Ch 9.2](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html)
*   **The Option Enum**: [The Rust Book Ch 6.1](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html?highlight=Option#the-option-enum-and-its-advantages-over-null-values)
*   **Rust By Example (Error Handling)**: [Error Handling](https://doc.rust-lang.org/rust-by-example/error.html)
*   **Rust Pattern (Result/Option)**: [Idioms](https://rust-unofficial.github.io/patterns/idioms/error-handling.html)
