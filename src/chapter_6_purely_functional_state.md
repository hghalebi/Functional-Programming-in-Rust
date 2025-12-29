# Chapter 6: Agent Memory (Purely Functional State)

Handling state without side effects is a core aspect of Agentic programming. Agents need to manage **Memory** (History, Context, Variables) as they execute steps. We use the **State Monad** pattern, where a state transition is a function `Memory -> (Action, Memory)`.

## 6.1 Generating random tokens (MockLLM)

To test our Agents deterministically, we need a Mock LLM that generates "Random" tokens but in a reproducible way.

```rust
pub trait MockLLM {
    fn next_token(&self) -> (i32, Box<dyn MockLLM>);
}

#[derive(Clone)]
pub struct SimpleMockLLM {
    seed: i64,
}

impl SimpleMockLLM {
    pub fn new(seed: i64) -> SimpleMockLLM {
        SimpleMockLLM { seed }
    }
}
# impl MockLLM for SimpleMockLLM {
#     fn next_token(&self) -> (i32, Box<dyn MockLLM>) {
#         let new_seed = (self.seed.wrapping_mul(0x5DEECE66D).wrapping_add(0xB)) & 0xFFFFFFFFFFFF;
#         let next_rng = SimpleMockLLM { seed: new_seed };
#         let n = (new_seed >> 16) as i32;
#         (n, Box::new(next_rng))
#     }
# }
# fn main() {
#     let llm = SimpleMockLLM::new(42);
#     let (token1, _next_state) = llm.next_token();
#     println!("Generated token ID: {}", token1);
# }
```

### Exercise 6.1: non_negative_token
### Exercise 6.2: double_precision_prob
### Exercise 6.3: int_double, double_int, double3
### Exercise 6.4: generate_token_list

## 6.4 A better API for state actions

We define `Sampler<A>` as a type alias for `Fn(MockLLM) -> (A, MockLLM)`. This represents a probabilistic action.

```rust
# pub trait MockLLM {}
type Sampler<A> = Box<dyn Fn(Box<dyn MockLLM>) -> (A, Box<dyn MockLLM>)>;
# fn main() {}
```

### Exercise 6.5: double via map
### Exercise 6.6: map2
### Exercise 6.7: sequence
### Exercise 6.8: flat_map
### Exercise 6.9: map and map2 via flat_map

## 6.5 A general Agent State generic

We generalize `Sampler` to `Agent<Memory, Action>`. An Agent is simply a function that takes a memory state and returns an action (decision) plus the new memory state.

```rust
// Agent<Memory, Action>
pub struct Agent<S, A>(pub Box<dyn Fn(S) -> (A, S)>);
# fn main() {}
```

### Exercise 6.10: unit, map, map2, flat_map, sequence for Agent

## 6.6 Purely functional imperative programming

Using `Agent` (State Monad), we can write imperative-looking code using `flat_map` chains to represent an Agent's Thought Process.

### Exercise 6.11: ChatBot State Machine
Implement a finite state automaton for a simple ChatBot.

```rust
pub enum Input {
    UserMessage,
    ModelCompletion,
}

pub struct ChatBot {
    pub awaiting_input: bool,
    pub context_tokens: i32,
    pub total_messages: i32,
}
# fn main() {}
```

Rules:
1. `UserMessage` while `awaiting_input` -> Transition to `!awaiting_input` (Generating).
2. `ModelCompletion` while `!awaiting_input` -> Transition to `awaiting_input` (Ready).
3. `UserMessage` while Generating (busy) is ignored.
4. `ModelCompletion` while Ready (idle) is ignored.

## 6.7 References

*   **Rust Book Ch 17 (State Object Pattern)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch17-03-oo-design-patterns.html)
*   **Rust By Example (Traits)**: [Traits](https://doc.rust-lang.org/rust-by-example/trait.html)
*   **Refactoring.Guru**: [State Pattern](https://refactoring.guru/design-patterns/state)
