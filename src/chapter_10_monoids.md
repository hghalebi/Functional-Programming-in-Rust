# Chapter 10: Monoids

In this chapter, we explore one of the simplest and most ubiquitous algebraic structures: the **Monoid**. This serves as our introduction to purely algebraic structures, defined solely by their operations and laws.

## 10.1 What is a Monoid?

A monoid consists of:
1.  A type `A`.
2.  An associative binary operation `op(a1, a2): A`.
3.  An identity element `zero: A` such that `op(x, zero) == x` and `op(zero, x) == x`.

In Rust, we can define this as a trait. Since a single type can have multiple monoid instances (e.g., integers can be a monoid under addition or multiplication), we define the trait on a struct representing the monoid logic, rather than on the data type itself (using the "type class instance" pattern).

```rust
pub trait Monoid<A> {
    fn op(&self, a1: A, a2: A) -> A;
    fn zero(&self) -> A;
}
# fn main() {}
```

### Examples

**String Monoid**: Concatenation is associative, and the empty string is the identity.
```rust
# pub trait Monoid<A> { fn op(&self, a1: A, a2: A) -> A; fn zero(&self) -> A; }
pub struct StringMonoid;
impl Monoid<String> for StringMonoid {
    fn op(&self, a1: String, a2: String) -> String { a1 + &a2 }
    fn zero(&self) -> String { "".to_string() }
}
# fn main() {}
```

**Int Addition**:
```rust
# pub trait Monoid<A> { fn op(&self, a1: A, a2: A) -> A; fn zero(&self) -> A; }
pub struct IntAddition;
impl Monoid<i32> for IntAddition {
    fn op(&self, a1: i32, a2: i32) -> i32 { a1 + a2 }
    fn zero(&self) -> i32 { 0 }
}
# fn main() {}
```

## 10.2 Folding with Monoids

Monoids are intimately related to folding lists. If we have a list of values `[a, b, c]` and a monoid for that type, we can reduce the list to a single value by combining them: `op(a, op(b, c))`.

We can generalize this to `fold_map`. If we have a list of type `A`, and a function `f: A -> B` where `B` has a monoid instance, we can map every element to `B` and then combine them.

```rust
# pub trait Monoid<A> { fn op(&self, a1: A, a2: A) -> A; fn zero(&self) -> A; }
pub fn fold_map<A, B, M, F>(as_: Vec<A>, m: M, f: F) -> B 
where M: Monoid<B>, F: Fn(A) -> B {
    as_.into_iter().fold(m.zero(), |acc, x| m.op(acc, f(x)))
}
# fn main() {}
```

## 10.3 Associativity and Parallelism

The associativity law (`op(op(a, b), c) == op(a, op(b, c))`) means we don't have to fold sequentially from left to right. We can use a **balanced fold**, splitting the list in half, reducing each half recursively, and then combining the results. This enables parallelism.

```rust
# pub trait Monoid<A> { fn op(&self, a1: A, a2: A) -> A; fn zero(&self) -> A; }
pub fn fold_map_v<A, B, M, F>(v: &[A], m: &M, f: &F) -> B
where M: Monoid<B>, F: Fn(&A) -> B + Clone, B: Clone {
    if v.is_empty() {
        m.zero()
    } else if v.len() == 1 {
        f(&v[0])
    } else {
        let (left, right) = v.split_at(v.len() / 2);
        let lb = fold_map_v(left, m, f);
        let rb = fold_map_v(right, m, f);
        m.op(lb, rb)
    }
}
# fn main() {}
```

## 10.4 Example: Word Count

A practical use case for parallel folding is counting words in a large string. If we split a string in the middle, we might split a word. To handle this, we define a specialized monoid `WC` (Word Count).

```rust
pub enum WC {
    Stub(String), // Partial word
    Part(String, i32, String), // (Left partial, count of full words, Right partial)
}
# fn main() {}
```

The monoid operation combines these parts, merging adjacent stubs to form complete words if necessary.

```rust
# pub trait Monoid<A> { fn op(&self, a1: A, a2: A) -> A; fn zero(&self) -> A; }
# pub enum WC { Stub(String), Part(String, i32, String) }
# struct WCMonoid;
impl Monoid<WC> for WCMonoid {
    fn op(&self, a1: WC, a2: WC) -> WC {
        match (a1, a2) {
            (WC::Stub(c), WC::Stub(d)) => WC::Stub(c + &d),
            (WC::Stub(c), WC::Part(l, w, r)) => WC::Part(c + &l, w, r),
            // ... logic to merge parts ...
            _ => WC::Stub(String::new()), // Simplified for example
        }
    }
    // ...
#   fn zero(&self) -> WC { WC::Stub("".to_string()) }
}
# fn main() {}
```

By mapping every character to a `WC` (whitespace becomes `Part("", 0, "")`, non-whitespace becomes `Stub(c)`), we can use `fold_map_v` to count words in parallel.

## 10.5 Conclusion

Monoids provide a simple but powerful abstraction for combining values. Their laws ensure that operations like parallel folding are safe and correct. This sets the stage for more complex algebraic structures like Functors and Monads, which we will explore in the next chapters.
