# Chapter 8: Property-Based Testing

In this chapter, we design a library for **Property-Based Testing**, similar to QuickCheck or ScalaCheck. The goal is to separate the *specification* of program behavior from the *generation* of test cases.

Instead of writing individual test scenarios (e.g., "reversing [1, 2, 3] yields [3, 2, 1]"), we define properties that should hold for *all* inputs (e.g., "reversing a list twice yields the original list"). The library then automatically generates random inputs to verify these properties.

## 8.1 Data Types for Testing

We need two core data types:
1.  **`Gen<A>`**: A generator that knows how to produce values of type `A`.
2.  **`Prop`**: A property that can be checked, resulting in success or failure.

### 8.1.1 Generators (`Gen`)

A generator is essentially a state transition over a random number generator. In Chapter 6, we built `State<RNG, A>`. `Gen<A>` is simply a wrapper around this state.

```rust
# struct State<S, A>(Box<dyn Fn(S) -> (A, S)>);
# struct SimpleRNG;
pub struct Gen<A>(State<SimpleRNG, A>);
# fn main() {}
```

This allows us to leverage all the combinators we wrote for `State`, like `map` and `flat_map`, to compose complex generators from simple ones.

#### Primitives
-   **`choose(start, stop)`**: Generates an integer in a range.
-   **`unit(a)`**: Always generates the value `a`.
-   **`boolean()`**: Generates a random boolean.

#### Combinators
-   **`list_of_n(n, gen)`**: Generates a list of length `n` using the given generator.
-   **`flat_map`**: Allows a generator to depend on the output of another generator.

### 8.1.2 Properties (`Prop`)

A property is something we can check. A simple boolean `check` is insufficient because we want to know *why* it failed (the counter-example) and how many tests passed.

Our `Prop` type encapsulates a function that takes configuration (max size, number of test cases) and a source of randomness, returning a `Result`.

```rust
# use std::rc::Rc;
# struct MaxSize(i32);
# struct TestCases(i32);
# struct SimpleRNG;
# struct FailedCase(String);
# struct SuccessCount(i32);
pub struct Prop(Rc<dyn Fn(MaxSize, TestCases, SimpleRNG) -> Result>);

pub enum Result {
    Passed,
    Falsified(FailedCase, SuccessCount),
    Proved,
}
# fn main() {}
```

We use `Rc<dyn Fn...>` to allow properties to be cloned and shared, which is essential when combining them (e.g., `prop1.and(prop2)`).

## 8.2 Test Case Minimization

Debugging failures on massive inputs is hard. There are two main approaches to minimization:
1.  **Shrinking**: Find a failure, then iteratively "shrink" the input to find a smaller failing case. (Used by QuickCheck/ScalaCheck).
2.  **Sized Generation**: Start with small inputs and gradually increase size. The first failure found is naturally small. (Used to some extent in Hedgehog).

We implement **Sized Generation** via `SGen`:

```rust
# use std::rc::Rc;
# struct Gen<A>(A); // Mock
pub struct SGen<A>(Rc<dyn Fn(i32) -> Gen<A>>);
# fn main() {}
```

`SGen` allows us to define generators that adapt to a requested size. `Prop` then iterates through sizes up to a maximum, ensuring we test small cases first.

## 8.3 The Rust Implementation

### Ownership and Closures
Unlike Scala, Rust requires explicit management of ownership.
-   **`Gen`**: Wraps `State`, which uses `Rc` internally in our Ch 6 implementation. This makes `Gen` cheap to clone.
-   **`Prop`**: Uses `Rc<dyn Fn...>` to store the check logic.
-   **Laziness**: In Scala, `check(true)` is lazy by name. In Rust, we pass a closure `check(|| true)` to prevent immediate execution.

### Random Number Generation
We reuse `SimpleRNG` from Chapter 6. A key challenge in Rust is that `RNG` is a trait. Passing `dyn RNG` around can be tricky due to object safety rules (especially if methods return `Box<dyn RNG>`). For this chapter, we specialized our implementation to use `SimpleRNG` directly to keep the focus on the property testing logic rather than generic trait juggling.

## 8.4 Example: Verifying `List::reverse`

```rust
# struct SGen<A>(A);
# impl<A> SGen<A> { fn list_of(g: Gen<A>) -> SGen<Vec<A>> { SGen(Vec::new()) } }
# struct Gen<A>(A);
# impl<A> Gen<A> { fn choose(start: i32, stop: i32) -> Gen<i32> { Gen(0) } }
# struct SimpleRNG; impl SimpleRNG { fn new(s: i64) -> Self { SimpleRNG } }
# struct Result; impl Result { fn is_falsified(&self) -> bool { false } }
# struct Prop; impl Prop { fn check(&self, a: i32, b: i32, rng: SimpleRNG) -> Result { Result } }
# fn for_all_sgen<A, F>(g: SGen<A>, f: F) -> Prop where F: Fn(A) -> bool { Prop }

#[test]
fn test_list_reverse() {
    let small_int = Gen::<i32>::choose(0, 100);
    let reverse_prop = for_all_sgen(SGen::list_of(small_int), |ns: Vec<i32>| {
        let mut rev = ns.clone();
        rev.reverse();
        let mut revrev = rev.clone();
        revrev.reverse();
        ns == revrev
    });
    // Run the check
    let result = reverse_prop.check(10, 100, SimpleRNG::new(42));
    assert!(!result.is_falsified());
}
# fn main() {} // Tests don't run in main usually but good to have
```

This ensures that for any generated list of integers, reversing it twice yields the original list. If we tried this with a broken reverse implementation, `Prop` would find a counter-example (e.g., `[0, 1]`) and report it.

## 8.5 Summary
We have built a powerful testing library from scratch. It demonstrates how functional design principles (composition, pure functions) apply to domain-specific languages like testing frameworks.

## 8.6 References

*   **Rust Book Ch 11 (Testing)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch11-00-testing.html)
*   **Rust By Example (Testing)**: [Testing](https://doc.rust-lang.org/rust-by-example/testing.html)
*   **Proptest (Crate)**: [Official Book](https://altsysrq.github.io/proptest-book/intro.html) - Real-world Property Testing in Rust.
