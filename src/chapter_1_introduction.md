# Chapter 1: What is functional programming?

Functional programming (FP) is based on a simple premise with far-reaching implications: we construct our programs using only pure functions—in other words, functions that have no side effects. What are side effects? A function has a side effect if it does something other than simply return a result, for example:

- Modifying a variable
- Modifying a data structure in place
- Setting a field on an object
- Throwing an exception or halting with an error
- Printing to the console or reading user input
- Reading from or writing to a file
- Drawing on the screen
- **Calling an external API (like an LLM or Database)**

We’ll provide a more precise definition of side effects later in this chapter, but consider what programming would be like without the ability to do these things. It may be difficult to imagine. How is it even possible to write useful Agentic workflows at all? If we can't call APIs or update memory, how do our Agents do anything?

The answer is that functional programming is a restriction on *how* we write programs, but not on *what* programs we can express. Over the course of this book, we’ll learn how to express all of our Agentic workflows without side effects, and that includes programs that perform I/O, handle errors, and manage context. We’ll learn how following the discipline of FP is tremendously beneficial because of the increase in modularity that we gain from programming with pure functions (which we can think of as **Deterministic Tools**). Because of their modularity, these tools are easier to test, reuse, parallelize, generalize, and reason about. Furthermore, pure functions are much less prone to bugs (or hallucinations in logic).

In this chapter, we’ll look at a simple program with side effects and demonstrate some of the benefits of FP by removing these side effects. We’ll also discuss the benefits of FP more generally and define two important concepts—referential transparency and the substitution model.

## 1.1 The benefits of FP: a simple example

Let’s look at an example that demonstrates some of the benefits of programming with pure functions. The point here is just to illustrate some basic ideas that we’ll return to throughout this book. This will also be your first exposure to Rust's syntax if you are new to the language.

### 1.1.1 A program with side effects

Suppose we’re implementing a client to interact with an Large Language Model (LLM). We want to generate text completions and bill our users for the tokens they consume. We’ll begin with a Rust program that uses side effects in its implementation (also called an impure program).

```rust
# #[derive(Clone, Copy)] struct Completion { tokens: i32 }
# impl Completion { fn new() -> Self { Completion { tokens: 100 } } }
# struct BillingService;
# impl BillingService { fn charge_usage(&mut self, _tokens: i32) {} }
struct LLMClient;

impl LLMClient {
    fn generate_completion(&self, billing: &mut BillingService) -> Completion {
        let response = Completion::new();
        billing.charge_usage(response.tokens);
        response
    }
}
# fn main() {
#     let client = LLMClient;
#     let mut billing = BillingService;
#     let _ = client.generate_completion(&mut billing);
# }
```

The line `billing.charge_usage(response.tokens)` is an example of a side effect. Charging for usage involves interaction with the outside world—suppose it requires contacting a database, updating a distributed quota system, or calling an external payment provider like Stripe.

But our function merely returns a `Completion` (the text response) and these other actions are happening on the side, hence the term “side effect.”

As a result of this side effect, the code is difficult to test. We don’t want our unit tests to actually contact the billing system and charge real money! This lack of testability is suggesting a design change: arguably, `LLMClient` shouldn’t have any knowledge baked into it about how to contact the billing system, nor should it have knowledge of how to persist transaction records. We can make the code more modular and testable by letting `LLMClient` be ignorant of these concerns and passing a `Billing` object into `generate_completion`.

```rust
# #[derive(Clone, Copy)] struct Completion { tokens: i32 }
# impl Completion { fn new() -> Self { Completion { tokens: 100 } } }
# struct BillingService;
# trait Billing { fn charge_usage(&mut self, tokens: i32); }
# struct MockBilling;
# impl Billing for MockBilling { fn charge_usage(&mut self, _tokens: i32) {} }
struct LLMClient;

impl LLMClient {
    fn generate_completion(&self, billing: &mut dyn Billing) -> Completion {
        let response = Completion::new();
        billing.charge_usage(response.tokens);
        response
    }
}
# fn main() {
#     let client = LLMClient;
#     let mut mock = MockBilling;
#     let _ = client.generate_completion(&mut mock);
# }
```

Though side effects still occur when we call `billing.charge_usage`, we have at least regained some testability. `Billing` can be a trait (interface), and we can write a mock implementation of this trait that is suitable for testing. But that isn’t ideal either. We’re forced to make `Billing` a trait, when a concrete struct may have been fine otherwise, and any mock implementation will be awkward to use. For example, it might contain some internal state that we’ll have to inspect after the call to `generate_completion`, and our test will have to make sure this state has been appropriately modified (mutated) by the call.

Separate from the concern of testing, there’s another problem: it’s difficult to reuse `generate_completion`. Suppose we want to implement a "Chain of Thought" workflow where we generate 12 intermediate reasoning steps. Ideally we could just reuse `generate_completion` for this, perhaps calling it 12 times in a loop. But as it is currently implemented, that will involve contacting the billing system 12 times! That adds significant latency and load to our billing infrastructure.

### 1.1.2 A functional solution: removing the side effects

The functional solution is to eliminate side effects and have `generate_completion` return the usage cost as a value in addition to returning the `Completion`. The concerns of processing the cost by checking quotas, persisting records, and so on, will be handled elsewhere.

Here’s what a functional solution might look like in Rust:

```rust
# #[derive(Clone)] struct AccountId;
# #[derive(Clone, Copy)] struct Completion { tokens: i32 }
# impl Completion { fn new() -> Self { Completion { tokens: 100 } } }
# struct TokenUsage { account: AccountId, tokens: i32 }
# impl TokenUsage { fn new(account: AccountId, tokens: i32) -> Self { TokenUsage { account, tokens } } }
struct LLMClient;

impl LLMClient {
    fn generate_completion(&self, account: &AccountId) -> (Completion, TokenUsage) {
        let response = Completion::new();
        (response, TokenUsage::new(account.clone(), response.tokens))
    }
}
# fn main() {
#     let client = LLMClient;
#     let account = AccountId;
#     let _ = client.generate_completion(&account);
# }
```

Here we’ve separated the concern of *creating* a usage record from the *processing* of that record. The `generate_completion` function now returns a `TokenUsage` as a value along with the `Completion`. We’ll see shortly how this lets us reuse it more easily to generate multiple completions with a single transaction. But what is `TokenUsage`? It’s a data type we just invented containing an `AccountId` and an amount, equipped with a handy function, `combine`, for combining usage logs for the same Account:

```rust
# #[derive(PartialEq, Clone)] struct AccountId;
struct TokenUsage {
    account: AccountId,
    tokens: i32,
}

impl TokenUsage {
    fn combine(&self, other: &TokenUsage) -> Result<TokenUsage, String> {
        if self.account == other.account {
            Ok(TokenUsage {
                account: self.account.clone(),
                tokens: self.tokens + other.tokens,
            })
        } else {
            Err("Can't combine usage for different accounts".to_string())
        }
    }
}
# fn main() {
#    let acc = AccountId;
#    let u1 = TokenUsage { account: acc.clone(), tokens: 50 };
#    let u2 = TokenUsage { account: acc, tokens: 150 };
#    assert!(u1.combine(&u2).is_ok());
# }
```

Now let’s look at `batch_generate`, to implement the generation of `n` completions. Unlike before, this can now be implemented in terms of `generate_completion`, as we had hoped.

```rust
# #[derive(PartialEq, Clone)] struct AccountId;
# #[derive(Clone, Copy)] struct Completion { tokens: i32 }
# impl Completion { fn new() -> Self { Completion { tokens: 100 } } }
# #[derive(Clone)] struct TokenUsage { account: AccountId, tokens: i32 }
# impl TokenUsage { 
#    fn new(account: AccountId, tokens: i32) -> Self { TokenUsage { account, tokens } } 
#    fn combine(&self, other: &TokenUsage) -> Result<TokenUsage, String> { Ok(TokenUsage { account: self.account.clone(), tokens: self.tokens + other.tokens }) }
# }
# struct LLMClient;
impl LLMClient {
#   fn generate_completion(&self, account: &AccountId) -> (Completion, TokenUsage) { (Completion::new(), TokenUsage::new(account.clone(), 100)) }
    fn batch_generate(&self, account: &AccountId, n: usize) -> (Vec<Completion>, TokenUsage) {
        let results: Vec<(Completion, TokenUsage)> = (0..n)
            .map(|_| self.generate_completion(account))
            .collect();
        
        let (completions, usages): (Vec<Completion>, Vec<TokenUsage>) = results.into_iter().unzip();
        
        let total_usage = usages.into_iter()
            .reduce(|u1, u2| u1.combine(&u2).unwrap())
            .unwrap_or(TokenUsage::new(account.clone(), 0));
            
        (completions, total_usage)
    }
}
# fn main() {
#     let client = LLMClient;
#     let acc = AccountId;
#     let (_, usage) = client.batch_generate(&acc, 12);
#     assert_eq!(usage.tokens, 1200);
# }
```

Overall, this solution is a marked improvement—we’re now able to reuse `generate_completion` directly to define the `batch_generate` function, and both functions are trivially testable without having to define complicated mock implementations of some `Billing` interface!

Making `TokenUsage` into a first-class value has other benefits we might not have anticipated: we can more easily assemble business logic for working with these usage logs. For instance, an Agent might run a complex multi-step workflow involving creating documents, researching, and coding. It might be nice if we could combine all the usage across these different tools into a single billable event. Since `TokenUsage` is first-class, we can write the following function to coalesce any same-account usage in a `Vec<TokenUsage>`:

```rust
# use std::collections::HashMap;
# #[derive(PartialEq, Eq, Clone, Hash)] struct AccountId;
# #[derive(Clone)] struct TokenUsage { account: AccountId, tokens: i32 }
# impl TokenUsage { fn combine(&self, other: &TokenUsage) -> Result<TokenUsage, String> { Ok(TokenUsage { account: self.account.clone(), tokens: self.tokens + other.tokens }) } }
 
fn coalesce_usage(logs: Vec<TokenUsage>) -> Vec<TokenUsage> {
    // In a real Rust app, we might use itertools using a HashMap or sort first
    // Since we don't have itertools in pure std, we can implement it simply.
    
    let mut groups: HashMap<AccountId, Vec<TokenUsage>> = HashMap::new();
    for log in logs {
        groups.entry(log.account.clone()).or_default().push(log);
    }
    
    groups.values()
        .map(|list| list.iter().cloned().reduce(|u1, u2| u1.combine(&u2).unwrap()).unwrap())
        .collect()
}
# fn main() {
#    let acc = AccountId;
#    let logs = vec![TokenUsage { account: acc.clone(), tokens: 10 }, TokenUsage { account: acc, tokens: 20 }];
#    let summary = coalesce_usage(logs);
#    assert_eq!(summary[0].tokens, 30);
# }
```

This sort of transformation can be applied to any function with side effects to push these effects to the outer layers of the program. Functional programmers often speak of implementing programs with a pure core and a thin layer on the outside that handles effects (I/O).

## 1.2 Exactly what is a (pure) function?

We said earlier that FP means programming with pure functions, and a pure function is one that lacks side effects. In our discussion of the LLM example, we worked off an informal notion of side effects and purity. Here we’ll formalize this notion, to pinpoint more precisely what it means to program functionally (or "Agentically").

A function `f` with input type `A` and output type `B` (written in Rust as `fn(A) -> B`) is a computation that relates every value `a` of type `A` to exactly one value `b` of type `B` such that `b` is determined solely by the value of `a`. Any changing state of an internal or external process is irrelevant to computing the result `f(a)`. For example, a function `int_to_string` having type `fn(i32) -> String` will take every integer to a corresponding string. Furthermore, if it really is a function, it will do nothing else.

In other words, a function (or **Deterministic Tool**) has no observable effect on the execution of the program other than to compute a result given its inputs; we say that it has no side effects.

We can formalize this idea of pure functions using the concept of **referential transparency (RT)**. This is a property of expressions in general and not just functions. For the purposes of our discussion, consider an expression to be any part of a program that can be evaluated to a result. For example, `2 + 3` is an expression that applies the pure function `+` to the values `2` and `3`. This has no side effect. The evaluation of this expression results in the same value `5` every time. In fact, if we saw `2 + 3` in a program we could simply replace it with the value `5` and it wouldn’t change a thing about the meaning of our program.

This is all it means for an expression to be referentially transparent—in any program, the expression can be replaced by its result without changing the meaning of the program. And we say that a function is pure if calling it with RT arguments is also RT.

## 1.3 Referential transparency, purity, and the substitution model

Let’s see how the definition of RT applies to our original `generate_completion` example:

```rust,ignore
fn generate_completion(&self, billing: &mut BillingService) -> Completion {
    let response = Completion::new();
    billing.charge_usage(response.tokens);
    response
}
```

Whatever the return type of `billing.charge_usage(...)` (perhaps it’s `()` unit), it’s discarded by `generate_completion`. Thus, the result of evaluating `generate_completion(my_account)` will be merely `response`. For `generate_completion` to be pure, by our definition of RT, it must be the case that `p(generate_completion(my_account))` behaves the same as `p(Completion::new())`, for any `p`.

This clearly doesn’t hold—the program `Completion::new()` doesn’t do anything, whereas `generate_completion(my_account)` will contact the billing system and charge real money. Already we have an observable difference between the two programs.

Referential transparency forces the invariant that everything a function does is represented by the value that it returns, according to the result type of the function. This constraint enables a simple and natural mode of reasoning about program evaluation called the **substitution model**. When expressions are referentially transparent, we can imagine that computation proceeds much like we’d solve an algebraic equation. We fully expand every part of an expression, replacing all variables with their referents, and then reduce it to its simplest form. At each step we replace a term with an equivalent one; computation proceeds by substituting equals for equals. In other words, RT enables equational reasoning about programs.

## 1.4 Summary

In this chapter, we introduced functional programming and explained exactly what FP is and why you might use it. We illustrated some of the benefits of FP using a simple example. We also discussed referential transparency and the substitution model and talked about how FP enables simpler reasoning about programs and greater modularity.

In this book, you’ll learn the concepts and principles of FP as they apply to every level of Agentic programming, starting from the simplest of tasks and building on that foundation.

## 1.5 References

*   **Rust Book Ch 1 (Introduction)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch01-00-getting-started.html)
*   **Rust By Example**: [Introduction](https://doc.rust-lang.org/rust-by-example/index.html)
