# Chapter 9: Parser Combinators

In this chapter, we explore the design of a combinator library for creating parsers. We'll use JSON parsing as our motivating use case, but the primary goal is to gain further insight into purely functional design. 

We will adopt an **algebraic design** approach. Instead of thinking about the internal representation of a parser first, we will start by designing the interface (the algebra)—the types and combinators we need—and specifying the laws they must satisfy. Only after we have a solid algebra will we worry about the concrete implementation.

## 9.1 Designing an Algebra

A parser is a program that takes unstructured input (like a string or stream of tokens) and produces a structured output (like a syntax tree or a specific value). In a combinator library, we build complex parsers by composing simpler ones.

Let's start with the basics. We need a type `Parser<A>` representing a parser that produces a value of type `A`. We also need a way to run it.

```rust
# struct ParseError;
pub trait Parser<A> {
    fn run(&self, input: &str) -> Result<A, ParseError>;
}
# fn main() {}
```

(Note: In our final implementation, `Parser` will be a struct wrapping a closure, but conceptually it's an interface provided by our library.)

### Basic Primitives

To parse a single character or string:

```rust
# struct Parser<A>(A);
pub fn char(c: char) -> Parser<char> { Parser(c) }
pub fn string(s: String) -> Parser<String> { Parser(s) }
# fn main() {}
```

We expect `run(char('a'), "a")` to return `Ok('a')`.

To combine parsers, we need an "or" combinator to choose between alternatives:

```rust
# struct Parser<A>(A);
pub fn or<A>(p1: Parser<A>, p2: Parser<A>) -> Parser<A> { p1 }
# fn main() {}
```

If `p1` fails, we try `p2`.

To handle repetition, we can introduce `many`:

```rust
# struct Parser<A>(A);
pub fn many<A>(p: Parser<A>) -> Parser<Vec<A>> { Parser(vec![]) }
# fn main() {}
```

And to transform the result of a parser, `map`:

```rust
# struct Parser<A>(A);
pub fn map<A, B, F>(p: Parser<A>, f: F) -> Parser<B> where F: Fn(A) -> B { Parser(f(p.0)) }
# fn main() {}
```

### Context Sensitivity and `flatMap`

Some parsing tasks require **context sensitivity**, where the next parser depends on the result of the previous one. For example, parsing a header that specifies the length of the following body. To support this, we need `flat_map`:

```rust
# struct Parser<A>(A);
pub fn flat_map<A, B, F>(p: Parser<A>, f: F) -> Parser<B> where F: Fn(A) -> Parser<B> { f(p.0) }
# fn main() {}
```

With `flat_map`, `map` and `product` (sequencing) can be derived.

## 9.2 Implementation

Now, let's look at how we can implement this algebra in Rust. Unlike Scala, where we might use trait inheritance heavily, purely functional parser combinators in Rust are often elegantly implemented using **functions** (closures).

### The Parser Type

A `Parser<A>` acts as a function `Fn(&Location) -> Result<(A, Location), ParseError>`. It takes a current input location and returns either a success (value + new location) or an error.

```rust
# use std::rc::Rc;
# struct Location;
# struct ParseError;
#[derive(Clone)]
#[allow(clippy::type_complexity)]
pub struct Parser<A>(Rc<dyn Fn(&Location) -> Result<(A, Location), ParseError>>);
# fn main() {}
```

We use `Rc<dyn Fn...>` to allow parsers to be cloned and shared easily, which is crucial for combinators that reuse parsers (like `many` reusing the same parser in a loop).

### Location and Error Handling

To provide good error messages, we track the `Location` in the input string.

```rust
# use std::rc::Rc;
#[derive(Debug, Clone, PartialEq)]
pub struct Location {
    pub input: Rc<String>,
    pub offset: usize,
}
# fn main() {}
```

A `ParseError` contains a stack of errors, allowing us to provide detailed context (e.g., "Error in JSON object at key 'foo': expected ':'").

```rust
# use std::rc::Rc;
# #[derive(Debug, Clone, PartialEq)] struct Location { input: Rc<String>, offset: usize }
#[derive(Debug, Clone, PartialEq)]
pub struct ParseError {
    pub stack: Vec<(Location, String)>,
}
# fn main() {}
```

### Combinators

The implementation of `string` checks if the input at the current location starts with the target string.

```rust
# use std::rc::Rc;
# #[derive(Debug, Clone, PartialEq)] struct Location { input: Rc<String>, offset: usize }
# impl Location { fn new(input: Rc<String>, offset: usize) -> Self { Location { input, offset } } }
# #[derive(Debug, Clone, PartialEq)] struct ParseError { stack: Vec<(Location, String)> }
# impl ParseError { fn new(loc: Location, msg: String) -> Self { ParseError { stack: vec![(loc, msg)] } } }
# struct Parser<A>(Rc<dyn Fn(&Location) -> Result<(A, Location), ParseError>>);
# impl<A> Parser<A> { fn new<F>(f: F) -> Self where F: Fn(&Location) -> Result<(A, Location), ParseError> + 'static { Parser(Rc::new(f)) } }
pub fn string(s: String) -> Parser<String> {
    Parser::new(move |loc: &Location| {
        let input_slice = &loc.input[loc.offset..];
        if input_slice.starts_with(&s) {
            Ok((s.clone(), Location::new(loc.input.clone(), loc.offset + s.len())))
        } else {
            Err(ParseError::new(loc.clone(), format!("Expected '{}'", s)))
        }
    })
}
# fn main() {}
```

The `or` combinator tries the first parser, and if it fails, tries the second.

The `flat_map` combinator runs the first parser, gets the result `a`, applies the function `f(a)` to get the second parser, and then runs that second parser.

## 9.3 Parsing JSON

With just a few primitives (`string`, `regex` or equivalent, `flat_map`, `or`, `many`), we can build a fully functional JSON parser.

We can define a `JSON` enum:

```rust
# use std::collections::HashMap;
pub enum JSON {
    JNull,
    JBool(bool),
    JNumber(f64),
    JString(String),
    JArray(Vec<JSON>),
    JObject(HashMap<String, JSON>),
}
# fn main() {}
```

And then compose our parsers to handle objects, arrays, and values recursively. Since JSON is recursive, we need to handle deferred execution properly (e.g., using `Parser::new` to wrap recursive calls).

## 9.4 Conclusion

In this chapter, we saw how algebraic design leads us to a flexible and powerful set of combinators. By focusing on the interface and laws first, we identified the core primitives (`flat_map`, `or`, `string`) needed to handle complex parsing tasks like JSON. The resulting implementation in Rust, leveraging closures and type-safe enums, is both expressive and efficient.
