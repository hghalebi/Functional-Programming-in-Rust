# Chapter 4: Handling errors without exceptions

Throwing exceptions is a side effect. In functional programming, we prefer to representing failures and exceptions with ordinary values. This preserves referential transparency.

In this chapter, we re-create two standard library types: `Option` and `Either`.

## 4.1 The Option data type

We represent the possibility of an undefined result with `Option`.

```rust
pub enum Option<A> {
    Some(A),
    None,
}
```

### Exercises 4.1: Basic Functions

Implement `map`, `flat_map`, `get_or_else`, `or_else`, and `filter`.

```rust
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
            Option::None => Option::None,
        }
    }

    pub fn get_or_else(self, default: A) -> A {
        match self {
            Option::Some(a) => a,
            Option::None => default,
        }
    }

    pub fn or_else<F>(self, ob: F) -> Option<A>
    where F: FnOnce() -> Option<A> {
        match self {
            Option::Some(_) => self,
            Option::None => ob(),
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
```

### Exercise 4.2: Variance
Implement `variance` in terms of `flat_map`.
Variance is the mean of `math.pow(x - m, 2)`.

```rust
fn mean(xs: &[f64]) -> Option<f64> {
    if xs.is_empty() {
        Option::None
    } else {
        Option::Some(xs.iter().sum::<f64>() / xs.len() as f64)
    }
}

pub fn variance(xs: &[f64]) -> Option<f64> {
    mean(xs).flat_map(|m| mean(&xs.iter().map(|x| (x - m).powi(2)).collect::<Vec<_>>()))
}
```

### Exercise 4.3: Map2
Combine two Option values.

```rust
pub fn map2<A, B, C, F>(a: Option<A>, b: Option<B>, f: F) -> Option<C>
where F: FnOnce(A, B) -> C {
    a.flat_map(|aa| b.map(|bb| f(aa, bb)))
}
```

### Exercise 4.4: Sequence
Combine a list of Options into one Option containing a list.

```rust
pub fn sequence<A>(a: Vec<Option<A>>) -> Option<Vec<A>> {
    let mut res = Vec::new();
    for opt in a {
        match opt {
            Option::Some(val) => res.push(val),
            Option::None => return Option::None,
        }
    }
    Option::Some(res)
}
```

### Exercise 4.5: Traverse
```rust
pub fn traverse<A, B, F>(a: Vec<A>, f: F) -> Option<Vec<B>>
where F: Fn(A) -> Option<B> {
    let mut res = Vec::new();
    for x in a {
        match f(x) {
            Option::Some(y) => res.push(y),
            Option::None => return Option::None,
        }
    }
    Option::Some(res)
}
```

## 4.4 The Either data type

`Option` doesn't tell us *why* something failed. `Either` lets us track a reason.

```rust
pub enum Either<E, A> {
    Left(E),
    Right(A),
}
```

### Exercise 4.6: Basic Functions on Either

```rust
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
            Either::Left(_) => b(),
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
```
*Note: In Rust, handling different error types `E` and `EE` usually requires `From` or a common Error trait.*

### Exercise 4.7: Sequence and Traverse for Either

```rust
pub fn sequence_either<E, A>(es: Vec<Either<E, A>>) -> Either<E, Vec<A>> {
    traverse_either(es, |x| x)
}

pub fn traverse_either<E, A, B, F>(as_vec: Vec<A>, f: F) -> Either<E, Vec<B>>
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
```

### Exercise 4.8: Accumulating Errors
To report *both* errors in `mkPerson` (name and age invalid), we would need a data structure that can hold multiple errors, like `Validation<Vec<E>, A>`. `Either` stops at the first error (fail-fast).

## 4.5 Summary
We learned to handle errors as values using `Option` and `Either`.

## 4.6 References

*   **Error Handling**: [The Rust Book Ch 9](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
*   **Recoverable Errors (Result)**: [The Rust Book Ch 9.2](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html)
*   **The Option Enum**: [The Rust Book Ch 6.1](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html?highlight=Option#the-option-enum-and-its-advantages-over-null-values)
*   **Rust By Example (Error Handling)**: [Error Handling](https://doc.rust-lang.org/rust-by-example/error.html)
*   **Rust Pattern (Result/Option)**: [Idioms](https://rust-unofficial.github.io/patterns/idioms/error-handling.html)
