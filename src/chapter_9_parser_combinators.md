# Chapter 9: Structured Output Parsing (Parser Combinators)

In this chapter, we explore the design of a **Structured Output Extractor** library. We'll use **JSON Parsing** as our motivating use case—specifically, extracting valid JSON from the often messy output of an LLM.

We will adopt an **algebraic design** approach. Instead of thinking about the internal regexes first, we will start by designing the interface (the algebra)—the types and combinators we need to reliably parse Tool Calls and Structured Data.

## 9.1 Designing an Algebra

An `Extractor` is a program that takes unstructured input (LLM text response) and produces a structured output (like a `ToolCall` struct). In a combinator library, we build complex extractors by composing simpler ones.

Let's start with the basics. We need a type `Extractor<A>` representing a parser that produces a value of type `A`.

```rust
# struct ParseError;
pub trait Extractor<A> {
    fn extract(&self, input: &str) -> Result<A, ParseError>;
}
# fn main() {}
```

(Note: In our final implementation, `Extractor` will be a struct wrapping a closure.)

### Basic Primitives

To parse a specific token (e.g. "json"):

```rust
# struct Extractor<A>(A);
pub fn token(s: String) -> Extractor<String> { Extractor(s) }
# fn main() {}
```

To combine parsers, we need an "or" combinator (Fallback strategy):

```rust
# struct Extractor<A>(A);
pub fn or<A>(p1: Extractor<A>, p2: Extractor<A>) -> Extractor<A> { p1 }
# fn main() {}
```

If `p1` fails (e.g. Model didn't use markdown code blocks), we try `p2` (e.g. Raw JSON).

To handle repetition (e.g. Multiple Tool Calls), we need `many`:

```rust
# struct Extractor<A>(A);
pub fn many<A>(p: Extractor<A>) -> Extractor<Vec<A>> { Extractor(vec![]) }
# fn main() {}
```

### Context Sensitivity and `flat_map`

Some parsing tasks require **context sensitivity**, where the next parser depends on the result of the previous one. For example, extracting a specific field based on a previously parsed "tool_name".

```rust
# struct Extractor<A>(A);
pub fn flat_map<A, B, F>(p: Extractor<A>, f: F) -> Extractor<B> where F: Fn(A) -> Extractor<B> { f(p.0) }
# fn main() {}
```

## 9.2 Implementation

Purely functional parser combinators in Rust are elegantly implemented using **functions** (closures).

### The Extractor Type

An `Extractor<A>` acts as a function `Fn(&Location) -> Result<(A, Location), ParseError>`.

```rust
use std::rc::Rc;

#[derive(Clone)]
#[allow(clippy::type_complexity)]
pub struct Extractor<A>(Rc<dyn Fn(&Location) -> Result<(A, Location), ParseError>>);
# struct Location;
# struct ParseError;
# fn main() {}
```

### Location and Error Handling

To provide good error messages (for "Refusal" or "Retry" prompts), we track the `Location`.

```rust
# use std::rc::Rc;
#[derive(Debug, Clone, PartialEq)]
pub struct Location {
    pub input: Rc<String>,
    pub offset: usize,
}
# fn main() {}
```

A `ParseError` contains a stack of errors, allowing us to feed this back to the LLM: "Error parsing JSON at index 45: expected ':'".

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

The implementation of `token` checks if the input at the current location starts with the target string.

```rust
# use std::rc::Rc;
# #[derive(Debug, Clone, PartialEq)] struct Location { input: Rc<String>, offset: usize }
# impl Location { fn new(input: Rc<String>, offset: usize) -> Self { Location { input, offset } } }
# #[derive(Debug, Clone, PartialEq)] struct ParseError { stack: Vec<(Location, String)> }
# impl ParseError { fn new(loc: Location, msg: String) -> Self { ParseError { stack: vec![(loc, msg)] } } }
# struct Extractor<A>(Rc<dyn Fn(&Location) -> Result<(A, Location), ParseError>>);
# impl<A> Extractor<A> { fn new<F>(f: F) -> Self where F: Fn(&Location) -> Result<(A, Location), ParseError> + 'static { Extractor(Rc::new(f)) } }
pub fn token(s: String) -> Extractor<String> {
    Extractor::new(move |loc: &Location| {
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

## 9.3 Parsing JSON from LLM Output

Most LLMs return JSON, but often wrapped in text. We need an extractor that can `skip_until` the first `{`.

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

## 9.4 Conclusion

In this chapter, we saw how algebraic design leads us to a flexible Extractor library. By focusing on primitives like `flat_map` and `or`, we can robustly handle the unpredictability of LLM textual output.
