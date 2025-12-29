# Chapter 8: Eval Benchmarks (Property-Based Testing)

In this chapter, we design a library for **Agent Evals**, similar to Property-Based Testing. The goal is to separate the *specification* of robust agent behavior from the *generation* of test cases.

Instead of writing individual test scenarios (e.g., "Ask about Paris weather"), we define **Invariants** that should hold for *all* inputs (e.g., "The response must always be valid JSON" or "The tool call must contain required arguments"). The library then automatically generates random inputs (Mock Prompts) to verify these properties.

## 8.1 Data Types for Evals

We need two core data types:
1.  **`TestCaseGenerator<A>`**: A tool that knows how to produce mock inputs of type `A` (e.g. valid User Queries, malicious prompt injections, etc.).
2.  **`EvalMetric`**: A property that can be checked, resulting in Pass/Fail.

### 8.1.1 Generators (`TestCaseGenerator`)

A generator is essentially a state transition over a random number generator (MockLLM).

```rust
# struct State<S, A>(Box<dyn Fn(S) -> (A, S)>);
# struct SimpleMockLLM;
pub struct TestCaseGenerator<A>(State<SimpleMockLLM, A>);
# fn main() {}
```

This allows us to leverage all the combinators we wrote for `Agent<S, A>`, like `map` and `flat_map`, to compose complex test cases.

#### Primitives
-   **`choose(start, stop)`**: Generates a random number (token).
-   **`weighted_boolean(p)`**: Generates true with probability `p` (e.g. simulate tool failure).

#### Combinators
-   **`list_of_n(n, gen)`**: Generates a conversation of length `n`.
-   **`flat_map`**: Allows a test case to depend on the output of another (e.g. generate a Tool Call, then generate a valid Output for it).

### 8.1.2 Properties (`EvalMetric`)

An eval metric is something we can check. A simple boolean `check` is insufficient because we want to know *why* it failed (the specific prompt that broke the agent) and how many tests passed.

```rust
# use std::rc::Rc;
# struct MaxSize(i32);
# struct TestCases(i32);
# struct SimpleMockLLM;
# struct FailedCase(String);
# struct SuccessCount(i32);
pub struct EvalMetric(Rc<dyn Fn(MaxSize, TestCases, SimpleMockLLM) -> EvalResult>);

pub enum EvalResult {
    Passed,
    Falsified(FailedCase, SuccessCount),
    Proved,
}
# fn main() {}
```

## 8.2 Test Case Minimization

Debugging agent failures on massive context windows is hard. There are two main approaches:
1.  **Shrinking**: Find a failure (long context), then iteratively remove messages to look for the root cause.
2.  **Sized Generation**: Start with short contexts and gradually increase size.

## 8.3 Example: Verifying `JsonTool` Robustness

Let's test if a `JsonTool` behaves correctly: "For any Valid JSON object, parsing it and then serializing it should yield the original object (semantically)."

```rust
# struct SGen<A>(A);
# impl<A> SGen<A> { fn list_of(g: TestCaseGenerator<A>) -> SGen<Vec<A>> { SGen(Vec::new()) } }
# struct TestCaseGenerator<A>(A);
# impl<A> TestCaseGenerator<A> { fn choose(start: i32, stop: i32) -> TestCaseGenerator<i32> { TestCaseGenerator(0) } }
# struct SimpleMockLLM; impl SimpleMockLLM { fn new(s: i64) -> Self { SimpleMockLLM } }
# struct EvalResult; impl EvalResult { fn is_falsified(&self) -> bool { false } }
# struct EvalMetric; impl EvalMetric { fn check(&self, a: i32, b: i32, rng: SimpleMockLLM) -> EvalResult { EvalResult } }
# fn for_all_sgen<A, F>(g: SGen<A>, f: F) -> EvalMetric where F: Fn(A) -> bool { EvalMetric }

#[test]
fn test_json_integrity() {
    let small_int = TestCaseGenerator::<i32>::choose(0, 100);
    // Pretend this generates JSON vectors
    let json_prop = for_all_sgen(SGen::list_of(small_int), |data: Vec<i32>| {
        let serialized = format!("{:?}", data);
        // Simulate parsing
        let parsed: Vec<i32> = serialized.trim_matches(|c| c == '[' || c == ']').split(", ").map(|s| s.parse().unwrap_or(0)).collect();
        data == parsed
    });
    
    // Run the eval
    let result = json_prop.check(10, 100, SimpleMockLLM::new(42));
    assert!(!result.is_falsified());
}
# fn main() {} 
```

This ensures that for any generated data structure, our "Agent" (Parser) preserves integrity.

## 8.5 Summary
We have built a powerful Eval framework. It demonstrates how functional design principles apply to verifying Agent behavior.

## 8.6 References

*   **Rust Book Ch 11 (Testing)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch11-00-testing.html)
*   **Rust By Example (Testing)**: [Testing](https://doc.rust-lang.org/rust-by-example/testing.html)
*   **Proptest (Crate)**: [Official Book](https://altsysrq.github.io/proptest-book/intro.html)
