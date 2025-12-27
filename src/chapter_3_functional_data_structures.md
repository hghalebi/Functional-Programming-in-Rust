# Chapter 3: Functional data structures

In this chapter, we’ll learn the concept of functional data structures and how to work with them. We’ll use this as an opportunity to introduce how data types are defined in functional programming, learn about the related technique of pattern matching, and get practice writing and generalizing pure functions.

## 3.1 Defining functional data structures

A functional data structure is operated on using only pure functions. Functional data structures are by definition immutable.

In Rust, the most ubiquitous functional data structure, the singly linked list, can be defined using an `enum`. To enable **data sharing** (persistence) as described in functional literature, we use `Rc` (Reference Counting) instead of `Box` (unique ownership). This allows multiple lists to share the same tail segments.

```rust
use std::rc::Rc;

#[derive(Clone, Debug)]
pub enum List<A> {
    Nil,
    Cons(A, Rc<List<A>>),
}

use List::*;

impl<A> Default for List<A> {
    fn default() -> Self {
        Nil
    }
}
```

Let’s look at the definition. `enum List` has two variants: `Nil` (empty) and `Cons` (non-empty). `Cons` holds a value of type `A` and a reference-counted pointer `Rc` to the rest of the list.

### Data Sharing
When we add an element `1` to the front of an existing list `xs`, we return a new list `Cons(1, Rc::new(xs))`. We don't copy `xs`; we just reuse it. This is called data sharing. Functional data structures are **persistent**, meaning existing references are never changed by operations.

## 3.2 Pattern matching

Rust supports pattern matching via `match`.

```rust
pub fn sum(ints: &List<i32>) -> i32 {
    match ints {
        Nil => 0,
        Cons(x, xs) => x + sum(xs),
    }
}

pub fn product(ds: &List<f64>) -> f64 {
    match ds {
        Nil => 1.0,
        Cons(0.0, _) => 0.0,
        Cons(x, xs) => x * product(xs),
    }
}
```

### Exercise 3.1
What will be the result of the match expression?
*Answer: The match expression in the book (translated to Rust syntax) would match the third case `x + y`, resulting in 3 (1 + 2).*

## 3.3 Data sharing in functional data structures

### Exercise 3.2: Tail
Implement the function `tail` for removing the first element of a List. Note that the function takes constant time.

```rust
pub fn tail<A>(l: &List<A>) -> Option<&Rc<List<A>>> {
    match l {
        Nil => None,
        Cons(_, xs) => Some(xs),
    }
}
```
*Note: In Rust, returning a reference to the tail `&Rc` works if we borrow the input. To return an owned persistent list, we would return `Rc<List<A>>` by cloning the pointer (cheap).*

### Exercise 3.3: Set Head
Implement `set_head`.

```rust
pub fn set_head<A>(l: &List<A>, h: A) -> List<A> 
where A: Clone {
    match l {
        Nil => panic!("set_head on empty list"), // Or return Result
        Cons(_, xs) => Cons(h, xs.clone()),
    }
}
```

### Exercise 3.4: Drop
Generalize `tail` to `drop`.

```rust
pub fn drop<A>(l: &List<A>, n: usize) -> &List<A> {
    if n == 0 {
        return l;
    }
    match l {
        Nil => &Nil,
        Cons(_, xs) => drop(xs, n - 1),
    }
}
```

### Exercise 3.5: DropWhile
```rust
pub fn drop_while<A, F>(l: &List<A>, f: F) -> &List<A> 
where F: Fn(&A) -> bool {
    match l {
        Cons(h, t) if f(h) => drop_while(t, f),
        _ => l,
    }
}
```

### Exercise 3.6: Init
Implement a function `init` that returns a List consisting of all but the last element.
*Why can't this be constant time? Because we must rebuild the path to the new end.*

```rust
pub fn init<A: Clone>(l: &List<A>) -> List<A> {
    match l {
        Nil => panic!("init of empty list"),
        Cons(_, xs) if matches!(**xs, Nil) => Nil,
        Cons(h, xs) => Cons(h.clone(), Rc::new(init(xs))),
    }
}
```

## 3.4 Recursion over lists and generalizing to higher-order functions

### Exercise 3.7 - 3.15: Folds

**Exercise 3.9: Length using fold_right**
```rust
pub fn length<A>(l: &List<A>) -> usize {
    fold_right(l, 0, |_, acc| acc + 1)
}
```

**Exercise 3.10: Fold Left (Tail recursive)**
```rust
pub fn fold_left<A, B, F>(l: &List<A>, z: B, f: F) -> B 
where F: Fn(B, &A) -> B {
    match l {
        Nil => z,
        Cons(h, t) => fold_left(t, f(z, h), f),
    }
}
```

**Exercise 3.12: Reverse**
```rust
pub fn reverse<A: Clone>(l: &List<A>) -> List<A> {
    fold_left(l, Nil, |acc, h| Cons(h.clone(), Rc::new(acc)))
}
```

## 3.5 Trees

Algebraic data types can be used to define other data structures.

```rust
pub enum Tree<A> {
    Leaf(A),
    Branch(Box<Tree<A>>, Box<Tree<A>>),
}
```
*Note: For Trees, `Box` (unique ownership) is often sufficient unless we need DAGs or explicit sharing.*

### Exercise 3.25: Size
```rust
pub fn size<A>(t: &Tree<A>) -> usize {
    match t {
        Tree::Leaf(_) => 1,
        Tree::Branch(l, r) => 1 + size(l) + size(r),
    }
}
```

### Exercise 3.26: Maximum
```rust
pub fn maximum(t: &Tree<i32>) -> i32 {
    match t {
        Tree::Leaf(v) => *v,
        Tree::Branch(l, r) => maximum(l).max(maximum(r)),
    }
}
```

### Exercise 3.28: Map
```rust
pub fn map<A, B, F>(t: &Tree<A>, f: &F) -> Tree<B>
where F: Fn(&A) -> B {
    match t {
        Tree::Leaf(v) => Tree::Leaf(f(v)),
        Tree::Branch(l, r) => Tree::Branch(Box::new(map(l, f)), Box::new(map(r, f))),
    }
}
```

## 3.6 Summary
We introduced algebraic data types (ADTs), `List` and `Tree`, and higher-order functions like `map`, `fold`, and `filter`.

## 3.7 References

*   **Rust Book Ch 6 (Enums)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch06-00-enums.html)
*   **Rust Book Ch 15 (Smart Pointers/Box)**: [The Rust Programming Language](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html)
*   **Rust By Example (Enums)**: [Custom Types](https://doc.rust-lang.org/rust-by-example/custom_types/enum.html)
*   **Rust By Example (LinkedList)**: [Testcase: Linked List](https://doc.rust-lang.org/rust-by-example/custom_types/enum/testcase_linked_list.html)
