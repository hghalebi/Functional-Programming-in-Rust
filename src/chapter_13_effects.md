# Chapter 13: Tool Use (External I/O)

This chapter explores how to handle **External Tools** (like reading files, searching the web, or calling APIs) in a purely functional way. The core idea is to separate the **description** of the tool call (`ToolAction`) from its **execution**.

> [!NOTE]
> In idiomatic Rust, I/O is typically handled imperatively with `std::io` or asynchronously with `Future`. This chapter explores the *pure functional modeling* of Tool Use, which powers the design of Agent Runtimes.

## 13.1 The ToolAction Type

We can model an effectful tool call as a value that, when "run", performs the effect.

```rust
pub struct ToolAction<A>(Box<dyn FnOnce() -> A>);

impl<A: 'static> ToolAction<A> {
    pub fn new<F>(f: F) -> Self where F: FnOnce() -> A + 'static { 
        ToolAction(Box::new(f)) 
    }
    
    pub fn execute(self) -> A { (self.0)() }
    
    pub fn then_act<B: 'static, F>(self, f: F) -> ToolAction<B>
    where F: FnOnce(A) -> ToolAction<B> + 'static
    {
        ToolAction::new(move || f(self.execute()).execute())
    }
}
# fn main() {
#     let action = ToolAction::new(|| 42);
#     assert_eq!(action.execute(), 42);
# }
```

## 13.2 Composition

Since `ToolAction` is a description, we can compose these descriptions. `ToolAction` forms a Monad.

- **`map`**: Transform the result (e.g., parse tool output).
- **`then_act` (flat_map)**: Chain tools, where the second tool depends on the result of the first.

```rust
# struct ToolAction<A>(Box<dyn FnOnce() -> A>);
# impl<A: 'static> ToolAction<A> {
#     fn new<F>(f: F) -> Self where F: FnOnce() -> A + 'static { ToolAction(Box::new(f)) }
#     fn execute(self) -> A { (self.0)() }
#     fn then_act<B: 'static, F>(self, f: F) -> ToolAction<B> where F: FnOnce(A) -> ToolAction<B> + 'static {
#         ToolAction::new(move || f((self.0)()).execute())
#     }
# }
struct Terminal;
impl Terminal {
    fn read_user_input() -> ToolAction<String> { ToolAction::new(|| "User query".to_string()) }
    fn emit_log(msg: &str) -> ToolAction<()> { ToolAction::new(|| ()) }
}

let agent_loop = Terminal::read_user_input()
    .then_act(|query| Terminal::emit_log(&format!("Processing: {}", query)));
# fn main() {}
```

Until we call `agent_loop.execute()`, nothing happens. This is **Referential Transparency**: the expression `agent_loop` can be replaced by its definition without changing the outcome (because the outcome *is just a description*, not the side effect itself).

## 13.3 Limitations and The Free Monad

Our simple `ToolAction` uses the Rust call stack. Deeply recursive loops (infinite agents) will overflow.
We solve this using **Trampolining** or the **Free Monad** (Data as Control Flow).

## 13.4 Use Cases

This pattern allows us to:
1.  **Test Agents**: We can swap the "Interpreter" to mock tools without changing the Agent's logic.
2.  **Safe Refactoring**: We can refactor effectful sequences with confidence.
3.  **Audit Logs**: We can inspect the `ToolAction` before running it to ensure safety/alignment.
