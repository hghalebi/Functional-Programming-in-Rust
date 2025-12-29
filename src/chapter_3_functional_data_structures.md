# Chapter 3: Context Windows (Functional Data Structures)

In this chapter, we’ll learn the concept of functional data structures and how to work with them. In the world of Large Language Models (LLMs), managing the **Context Window** is critical. We often need efficient ways to manage conversation history, truncate old messages, or summarize dialogue without copying massive amounts of text.

We’ll use this as an opportunity to introduce how data types are defined in functional programming, learn about the related technique of pattern matching, and get practice writing and generalizing pure functions.

## 3.1 Defining functional data structures

A functional data structure is operated on using only pure functions. Functional data structures are by definition immutable. This is perfect for maintaining a **Message History** where we might want to fork a conversation (branching paths) without copying the entire shared history.

In Rust, the most ubiquitous functional data structure, the singly linked list, can be defined using an `enum`. To enable **data sharing** (persistence) as described in functional literature, we use `Rc` (Reference Counting) instead of `Box` (unique ownership). This allows multiple conversation branches to share the same tail segments.

```rust
use std::rc::Rc;

#[derive(Clone, Debug)]
pub enum List<A> {
    Nil,
    Cons(A, Rc<List<A>>),
}

use self::List::*;

impl<A> Default for List<A> {
    fn default() -> Self {
        Nil
    }
}
# fn main() {
#     let history: List<&str> = List::Cons("User: Hello", Rc::new(List::Nil));
#     println!("{:?}", history);
# }
```

Let’s look at the definition. `enum List` has two variants: `Nil` (empty) and `Cons` (non-empty). `Cons` holds a value of type `A` (a Message) and a reference-counted pointer `Rc` to the rest of the list (Previous history).

### Data Sharing
When we add a new message to the history, we return a new list `Cons(msg, Rc::new(history))`. We don't copy the old history; we just reuse it. This is called data sharing. Functional data structures are **persistent**, meaning existing references (older snapshots of the context) are never changed by operations.

## 3.2 Pattern matching

Rust supports pattern matching via `match`.

```rust
# use std::rc::Rc;
# enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn total_tokens(messages: &List<i32>) -> i32 {
    match messages {
        Nil => 0,
        Cons(tokens, rest) => tokens + total_tokens(rest),
    }
}

pub fn product_probabilities(probs: &List<f64>) -> f64 {
    match probs {
        Nil => 1.0,
        Cons(0.0, _) => 0.0, // Short-circuit if probability is 0
        Cons(p, rest) => p * product_probabilities(rest),
    }
}
# fn main() {
#     let history = Cons(100, Rc::new(Cons(200, Rc::new(Nil))));
#     assert_eq!(total_tokens(&history), 300);
# }
```

### Exercise 3.1
What will be the result of the match expression?
*Answer: The match expression in the book (translated to Rust syntax) would match the third case `x + y`, resulting in 3.*

## 3.3 Data sharing in functional data structures

### Exercise 3.2: Tail (Trim Oldest)
Implement the function `tail` for removing the first element ("trimming the most recent message" if defined as cons-stack, or "oldest" if defined as queue). In a List, `Cons` adds to the front. So `tail` removes the most recent addition.
*Note: In an immutable list, accessing the "rest" is O(1).*

```rust
# use std::rc::Rc;
# enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn tail<A>(l: &List<A>) -> Option<&Rc<List<A>>> {
    match l {
        Nil => None,
        Cons(_, xs) => Some(xs),
    }
}
# fn main() {
#    let list = Cons(1, Rc::new(Nil));
#    assert!(tail(&list).is_some());
# }
```
*Note: In Rust, returning a reference to the tail `&Rc` works if we borrow the input. To return an owned persistent list, we would return `Rc<List<A>>` by cloning the pointer (cheap).*

### Exercise 3.3: Set Head (Replace Most Recent Message)
Implement `set_head`. Useful for "Regenerating" the last response.

```rust
# use std::rc::Rc;
# #[derive(Debug)] enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn set_head<A>(l: &List<A>, h: A) -> List<A> 
where A: Clone {
    match l {
        Nil => panic!("set_head on empty history"), // Or return Result
        Cons(_, xs) => Cons(h, xs.clone()),
    }
}
# fn main() {
#    let list = Cons(1, Rc::new(Nil));
#    let new_list = set_head(&list, 2);
# }
```

### Exercise 3.4: Drop (Truncate N messages)
Generalize `tail` to `drop`.

```rust
# use std::rc::Rc;
# enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn drop<A>(l: &List<A>, n: usize) -> &List<A> {
    if n == 0 {
        return l;
    }
    match l {
        Nil => &Nil,
        Cons(_, xs) => drop(xs, n - 1),
    }
}
# fn main() {}
```

### Exercise 3.5: DropWhile (Truncate until condition)
Useful for "Remove messages until System Prompt".

```rust
# use std::rc::Rc;
# enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn drop_while<A, F>(l: &List<A>, f: F) -> &List<A> 
where F: Fn(&A) -> bool {
    match l {
        Cons(h, t) if f(h) => drop_while(t, f),
        _ => l,
    }
}
# fn main() {}
```

### Exercise 3.6: Init (Remove oldest message)
Implement a function `init` that returns a List consisting of all but the last element (the oldest message).
*Why can't this be constant time? Because in a singly linked list `Cons(A, Rest)`, the end is far away. We must rebuild the path.*

```rust
# use std::rc::Rc;
# enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn init<A: Clone>(l: &List<A>) -> List<A> {
    match l {
        Nil => panic!("init of empty list"),
        Cons(_, xs) if matches!(**xs, Nil) => Nil,
        Cons(h, xs) => Cons(h.clone(), Rc::new(init(xs))),
    }
}
# fn main() {}
```

## 3.4 Recursion over lists and generalizing to higher-order functions

### Exercise 3.7 - 3.15: Folds (Context Summarization)

Folding is the essence of **Summarization**. You take a list of messages and reduce them to a single value (a Summary).

**Exercise 3.9: Length (Message Count)**
```rust
# use std::rc::Rc;
# #[derive(Clone)] enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
# fn fold_right<A, B, F>(l: &List<A>, z: B, f: F) -> B where F: Fn(&A, B) -> B + Clone { match l { Nil => z, Cons(h, t) => f(h, fold_right(t, z, f.clone())) } } 
pub fn count_messages<A>(l: &List<A>) -> usize {
    fold_right(l, 0, |_, acc| acc + 1)
}
# fn main() {}
```

**Exercise 3.10: Fold Left (Iterative Summarization)**
Ideal for large context windows to avoid stack overflow.

```rust
# use std::rc::Rc;
# #[derive(Clone)] enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
pub fn fold_left<A, B, F>(l: &List<A>, z: B, f: F) -> B 
where F: Fn(B, &A) -> B {
    match l {
        Nil => z,
        Cons(h, t) => fold_left(t, f(z, h), f),
    }
}
# fn main() {}
```

**Exercise 3.12: Reverse (Chronological Sorting)**
Linked Lists are usually built in reverse order (stack). To get chronological order for the LLM, we reverse.

```rust
# use std::rc::Rc;
# #[derive(Clone)] enum List<A> { Nil, Cons(A, Rc<List<A>>) }
# use self::List::*;
# pub fn fold_left<A, B, F>(l: &List<A>, z: B, f: F) -> B where F: Fn(B, &A) -> B { match l { Nil => z, Cons(h, t) => fold_left(t, f(z, h), f), } }
pub fn reverse<A: Clone>(l: &List<A>) -> List<A> {
    fold_left(l, Nil, |acc, h| Cons(h.clone(), Rc::new(acc)))
}
# fn main() {}
```

## 3.5 Conversation Trees

Algebraic data types can be used to define other data structures. A **Conversation Tree** represents branching dialogue paths.

```rust
pub enum Tree<A> {
    Leaf(A),
    Branch(Box<Tree<A>>, Box<Tree<A>>),
}
# fn main() {}
```
*Note: For Trees, `Box` (unique ownership) is often sufficient unless we need DAGs or explicit sharing.*

### Exercise 3.25: Size (Total Turns)
```rust
# enum Tree<A> { Leaf(A), Branch(Box<Tree<A>>, Box<Tree<A>>) }
pub fn count_turns<A>(t: &Tree<A>) -> usize {
    match t {
        Tree::Leaf(_) => 1,
        Tree::Branch(l, r) => 1 + count_turns(l) + count_turns(r),
    }
}
# fn main() {}
```

### Exercise 3.26: Maximum (Max Token Count)
```rust
# enum Tree<A> { Leaf(A), Branch(Box<Tree<A>>, Box<Tree<A>>) }
pub fn max_token_usage(t: &Tree<i32>) -> i32 {
    match t {
        Tree::Leaf(v) => *v,
        Tree::Branch(l, r) => max_token_usage(l).max(max_token_usage(r)),
    }
}
# fn main() {}
```

### Exercise 3.28: Map (Sanitize Messages)
Apply a function (e.g., PII Redaction) to every node.

```rust
# enum Tree<A> { Leaf(A), Branch(Box<Tree<A>>, Box<Tree<A>>) }
pub fn map_conversation<A, B, F>(t: &Tree<A>, f: &F) -> Tree<B>
where F: Fn(&A) -> B {
    match t {
        Tree::Leaf(v) => Tree::Leaf(f(v)),
        Tree::Branch(l, r) => Tree::Branch(Box::new(map_conversation(l, f)), Box::new(map_conversation(r, f))),
    }
}
# fn main() {}
```

## 3.6 Summary
We introduced algebraic data types (ADTs), `List` (Message History) and `Tree` (Conversation Branches), and higher-order functions like `map`, `fold` (Summarize), and `filter`.

## 3.7 References

*   **Rust Book Ch 6 (Enums)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch06-00-enums.html)
*   **Rust Book Ch 15 (Smart Pointers/Box)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
*   **Rust By Example (Enums)**: [Custom Types](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html)
*   **Rust By Example (LinkedList)**: [Testcase: Linked List](https://doc.rust-lang.org/rust-by-example/custom_types/enum/testcase_linked_list.html)
