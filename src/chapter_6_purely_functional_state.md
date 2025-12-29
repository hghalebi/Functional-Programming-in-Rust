# Chapter 6: Purely Functional State

Handling state without side effects is a core aspect of functional programming. We use the **State Monad** pattern, where a state transition is a function `S -> (A, S)`.

## 6.1 Generating random numbers (Simple RNG)

```rust
pub trait RNG {
    fn next_int(&self) -> (i32, Box<dyn RNG>);
}

#[derive(Clone)]
pub struct SimpleRNG {
    seed: i64,
}

impl SimpleRNG {
    pub fn new(seed: i64) -> SimpleRNG {
        SimpleRNG { seed }
    }
}
# impl RNG for SimpleRNG {
#     fn next_int(&self) -> (i32, Box<dyn RNG>) {
#         let new_seed = (self.seed.wrapping_mul(0x5DEECE66D).wrapping_add(0xB)) & 0xFFFFFFFFFFFF;
#         let next_rng = SimpleRNG { seed: new_seed };
#         let n = (new_seed >> 16) as i32;
#         (n, Box::new(next_rng))
#     }
# }
# fn main() {
#     let rng = SimpleRNG::new(42);
#     let (n1, _rng2) = rng.next_int();
#     println!("{}", n1);
# }
```

### Exercise 6.1: non_negative_int
### Exercise 6.2: double
### Exercise 6.3: int_double, double_int, double3
### Exercise 6.4: ints

## 6.4 A better API for state actions

We define `Rand<A>` as a type alias for `Fn(RNG) -> (A, RNG)`.

```rust
# pub trait RNG {}
type Rand<A> = Box<dyn Fn(Box<dyn RNG>) -> (A, Box<dyn RNG>)>;
# fn main() {}
```

### Exercise 6.5: double via map
### Exercise 6.6: map2
### Exercise 6.7: sequence
### Exercise 6.8: flat_map
### Exercise 6.9: map and map2 via flat_map

## 6.5 A general State generic

We generalize `Rand` to `State<S, A>`.

```rust
pub struct State<S, A>(pub Box<dyn Fn(S) -> (A, S)>);
# fn main() {}
```

### Exercise 6.10: unit, map, map2, flat_map, sequence for State

## 6.6 Purely functional imperative programming

Using `State`, we can write imperative-looking code using `flat_map` chains.

### Exercise 6.11: Candy Machine
Implement a finite state automaton for a candy dispenser.

```rust
pub enum Input {
    Coin,
    Turn,
}

pub struct Machine {
    pub locked: bool,
    pub candies: i32,
    pub coins: i32,
}
# fn main() {}
```

3. Turn locked or Coin unlocked does nothing.
4. Machine out of candy ignores inputs.

## 6.7 References

*   **Rust Book Ch 17 (State Object Pattern)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch17-03-oo-design-patterns.html)
*   **Rust By Example (Traits)**: [Traits](https://doc.rust-lang.org/rust-by-example/trait.html)
*   **Refactoring.Guru**: [State Pattern](https://refactoring.guru/design-patterns/state)
