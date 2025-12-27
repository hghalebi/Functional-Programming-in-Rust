# Chapter 1: What is functional programming?

Functional programming (FP) is based on a simple premise with far-reaching implications: we construct our programs using only pure functions—in other words, functions that have no side effects. What are side effects? A function has a side effect if it does something other than simply return a result, for example:

- Modifying a variable
- Modifying a data structure in place
- Setting a field on an object
- Throwing an exception or halting with an error
- Printing to the console or reading user input
- Reading from or writing to a file
- Drawing on the screen

We’ll provide a more precise definition of side effects later in this chapter, but consider what programming would be like without the ability to do these things, or with significant restrictions on when and how these actions can occur. It may be difficult to imagine. How is it even possible to write useful programs at all? If we can’t reassign variables, how do we write simple programs like loops? What about working with data that changes, or handling errors without throwing exceptions? How can we write programs that must perform I/O, like drawing to the screen or reading from a file?

The answer is that functional programming is a restriction on how we write programs, but not on what programs we can express. Over the course of this book, we’ll learn how to express all of our programs without side effects, and that includes programs that perform I/O, handle errors, and modify data. We’ll learn how following the discipline of FP is tremendously beneficial because of the increase in modularity that we gain from programming with pure functions. Because of their modularity, pure functions are easier to test, reuse, parallelize, generalize, and reason about. Furthermore, pure functions are much less prone to bugs.

In this chapter, we’ll look at a simple program with side effects and demonstrate some of the benefits of FP by removing these side effects. We’ll also discuss the benefits of FP more generally and define two important concepts—referential transparency and the substitution model.

## 1.1 The benefits of FP: a simple example

Let’s look at an example that demonstrates some of the benefits of programming with pure functions. The point here is just to illustrate some basic ideas that we’ll return to throughout this book. This will also be your first exposure to Rust's syntax if you are new to the language.

### 1.1.1 A program with side effects

Suppose we’re implementing a program to handle purchases at a coffee shop. We’ll begin with a Rust program that uses side effects in its implementation (also called an impure program).

```rust,ignore
struct Cafe;

impl Cafe {
    fn buy_coffee(&self, cc: &mut CreditCard) -> Coffee {
        let cup = Coffee::new();
        cc.charge(cup.price);
        cup
    }
}
```

The line `cc.charge(cup.price)` is an example of a side effect. Charging a credit card involves some interaction with the outside world—suppose it requires contacting the credit card company via some web service, authorizing the transaction, charging the card, and (if successful) persisting some record of the transaction for later reference. 

But our function merely returns a `Coffee` and these other actions are happening on the side, hence the term “side effect.”

As a result of this side effect, the code is difficult to test. We don’t want our tests to actually contact the credit card company and charge the card! This lack of testability is suggesting a design change: arguably, `CreditCard` shouldn’t have any knowledge baked into it about how to contact the credit card company to actually execute a charge, nor should it have knowledge of how to persist a record of this charge in our internal systems. We can make the code more modular and testable by letting `CreditCard` be ignorant of these concerns and passing a `Payments` object into `buy_coffee`.

```rust,ignore
struct Cafe;

impl Cafe {
    fn buy_coffee(&self, cc: &mut CreditCard, p: &mut Payments) -> Coffee {
        let cup = Coffee::new();
        p.charge(cc, cup.price);
        cup
    }
}
```

Though side effects still occur when we call `p.charge(cc, cup.price)`, we have at least regained some testability. `Payments` can be a trait (interface), and we can write a mock implementation of this trait that is suitable for testing. But that isn’t ideal either. We’re forced to make `Payments` a trait, when a concrete struct may have been fine otherwise, and any mock implementation will be awkward to use. For example, it might contain some internal state that we’ll have to inspect after the call to `buy_coffee`, and our test will have to make sure this state has been appropriately modified (mutated) by the call to `charge`.

Separate from the concern of testing, there’s another problem: it’s difficult to reuse `buy_coffee`. Suppose a customer, Alice, would like to order 12 cups of coffee. Ideally we could just reuse `buy_coffee` for this, perhaps calling it 12 times in a loop. But as it is currently implemented, that will involve contacting the payment system 12 times, authorizing 12 separate charges to Alice’s credit card! That adds more processing fees and isn’t good for Alice or the coffee shop.

### 1.1.2 A functional solution: removing the side effects

The functional solution is to eliminate side effects and have `buy_coffee` return the charge as a value in addition to returning the `Coffee`. The concerns of processing the charge by sending it off to the credit card company, persisting a record of it, and so on, will be handled elsewhere.

Here’s what a functional solution might look like in Rust:

```rust,ignore
struct Cafe;

impl Cafe {
    fn buy_coffee(&self, cc: &CreditCard) -> (Coffee, Charge) {
        let cup = Coffee::new();
        (cup, Charge::new(cc.clone(), cup.price))
    }
}
```

Here we’ve separated the concern of creating a charge from the processing or interpretation of that charge. The `buy_coffee` function now returns a `Charge` as a value along with the `Coffee`. We’ll see shortly how this lets us reuse it more easily to purchase multiple coffees with a single transaction. But what is `Charge`? It’s a data type we just invented containing a `CreditCard` and an amount, equipped with a handy function, `combine`, for combining charges with the same `CreditCard`:

```rust
struct Charge {
    cc: CreditCard,
    amount: f64,
}

impl Charge {
    fn combine(&self, other: &Charge) -> Result<Charge, String> {
        if self.cc == other.cc {
            Ok(Charge {
                cc: self.cc.clone(),
                amount: self.amount + other.amount,
            })
        } else {
            Err("Can't combine charges to different cards".to_string())
        }
    }
}
```

Now let’s look at `buy_coffees`, to implement the purchase of `n` cups of coffee. Unlike before, this can now be implemented in terms of `buy_coffee`, as we had hoped.

```rust,ignore
impl Cafe {
    fn buy_coffees(&self, cc: &CreditCard, n: usize) -> (Vec<Coffee>, Charge) {
        let purchases: Vec<(Coffee, Charge)> = (0..n)
            .map(|_| self.buy_coffee(cc))
            .collect();
        
        let (coffees, charges): (Vec<Coffee>, Vec<Charge>) = purchases.into_iter().unzip();
        
        let combined_charge = charges.into_iter()
            .reduce(|c1, c2| c1.combine(&c2).unwrap())
            .unwrap_or(Charge::new(cc.clone(), 0.0)); // Handle 0 coffees case
            
        (coffees, combined_charge)
    }
}
```

Overall, this solution is a marked improvement—we’re now able to reuse `buy_coffee` directly to define the `buy_coffees` function, and both functions are trivially testable without having to define complicated mock implementations of some `Payments` interface!

Making `Charge` into a first-class value has other benefits we might not have anticipated: we can more easily assemble business logic for working with these charges. For instance, Alice may bring her laptop to the coffee shop and work there for a few hours, making occasional purchases. It might be nice if the coffee shop could combine these purchases Alice makes into a single charge, again saving on credit card processing fees. Since `Charge` is first-class, we can write the following function to coalesce any same-card charges in a `Vec<Charge>`:

```rust,ignore
fn coalesce(charges: Vec<Charge>) -> Vec<Charge> {
    // In a real Rust app, we might use itertools using a HashMap or sort first
    // Since we don't have itertools in pure std, we can implement it simply.
    // For brevity, let's assume we have a way to group by card:
    
    let mut groups: HashMap<CreditCard, Vec<Charge>> = HashMap::new();
    for charge in charges {
        groups.entry(charge.cc.clone()).or_default().push(charge);
    }
    
    groups.values()
        .map(|list| list.iter().cloned().reduce(|c1, c2| c1.combine(&c2).unwrap()).unwrap())
        .collect()
}
```

This sort of transformation can be applied to any function with side effects to push these effects to the outer layers of the program. Functional programmers often speak of implementing programs with a pure core and a thin layer on the outside that handles effects.

## 1.2 Exactly what is a (pure) function?

We said earlier that FP means programming with pure functions, and a pure function is one that lacks side effects. In our discussion of the coffee shop example, we worked off an informal notion of side effects and purity. Here we’ll formalize this notion, to pinpoint more precisely what it means to program functionally.

A function `f` with input type `A` and output type `B` (written in Rust as `fn(A) -> B`) is a computation that relates every value `a` of type `A` to exactly one value `b` of type `B` such that `b` is determined solely by the value of `a`. Any changing state of an internal or external process is irrelevant to computing the result `f(a)`. For example, a function `int_to_string` having type `fn(i32) -> String` will take every integer to a corresponding string. Furthermore, if it really is a function, it will do nothing else.

In other words, a function has no observable effect on the execution of the program other than to compute a result given its inputs; we say that it has no side effects.

We can formalize this idea of pure functions using the concept of **referential transparency (RT)**. This is a property of expressions in general and not just functions. For the purposes of our discussion, consider an expression to be any part of a program that can be evaluated to a result. For example, `2 + 3` is an expression that applies the pure function `+` to the values `2` and `3`. This has no side effect. The evaluation of this expression results in the same value `5` every time. In fact, if we saw `2 + 3` in a program we could simply replace it with the value `5` and it wouldn’t change a thing about the meaning of our program.

This is all it means for an expression to be referentially transparent—in any program, the expression can be replaced by its result without changing the meaning of the program. And we say that a function is pure if calling it with RT arguments is also RT.

## 1.3 Referential transparency, purity, and the substitution model

Let’s see how the definition of RT applies to our original `buy_coffee` example:

```rust,ignore
fn buy_coffee(&self, cc: &mut CreditCard) -> Coffee {
    let cup = Coffee::new();
    cc.charge(cup.price);
    cup
}
```

Whatever the return type of `cc.charge(cup.price)` (perhaps it’s `()` unit), it’s discarded by `buy_coffee`. Thus, the result of evaluating `buy_coffee(alice_credit_card)` will be merely `cup`, which is equivalent to a `Coffee::new()`. For `buy_coffee` to be pure, by our definition of RT, it must be the case that `p(buy_coffee(alice_credit_card))` behaves the same as `p(Coffee::new())`, for any `p`.

This clearly doesn’t hold—the program `Coffee::new()` doesn’t do anything, whereas `buy_coffee(alice_credit_card)` will contact the credit card company and authorize a charge. Already we have an observable difference between the two programs.

Referential transparency forces the invariant that everything a function does is represented by the value that it returns, according to the result type of the function. This constraint enables a simple and natural mode of reasoning about program evaluation called the **substitution model**. When expressions are referentially transparent, we can imagine that computation proceeds much like we’d solve an algebraic equation. We fully expand every part of an expression, replacing all variables with their referents, and then reduce it to its simplest form. At each step we replace a term with an equivalent one; computation proceeds by substituting equals for equals. In other words, RT enables equational reasoning about programs.

## 1.4 Summary

In this chapter, we introduced functional programming and explained exactly what FP is and why you might use it. We illustrated some of the benefits of FP using a simple example. We also discussed referential transparency and the substitution model and talked about how FP enables simpler reasoning about programs and greater modularity.

In this book, you’ll learn the concepts and principles of FP as they apply to every level of programming, starting from the simplest of tasks and building on that foundation.

## 1.5 References

*   **Rust Book Ch 1 (Introduction)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch01-00-getting-started.html)
*   **Rust By Example**: [Introduction](https://doc.rust-lang.org/rust-by-example/index.html)
