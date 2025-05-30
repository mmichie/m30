# M30 vs Rust: What's Different?

## You're Not Recreating Rust!

### Core Differences

| Feature | Rust | M30 |
|---------|------|-----|
| Memory Management | Manual (ownership/borrowing) | Garbage Collected |
| Syntax | C-like with braces | Python-like with indentation |
| Learning Curve | Steep (borrow checker) | Gentler (no lifetimes) |
| Mutability | Explicit `mut` everywhere | Immutable by default classes |
| Null Safety | No null (Option type) | No null (Option type) ✓ |
| Error Handling | Result type | Result type ✓ |
| Concurrency | Ownership prevents races | Structured concurrency |
| Type Inference | Local only | Islands (fuller inference) |
| Compile Target | Native only | Native + potential VM |

### What You're Actually Building

**A "Best of" Language:**
- Python's readability and syntax
- ML's type inference  
- Go's fast compilation and tooling
- Rust's error handling and enums
- Modern structured concurrency
- GC for simplicity

### Key Design Differences from Rust

#### 1. No Ownership/Borrowing
```rust
// Rust - must think about ownership
fn process(data: &mut Vec<i32>) { // borrowed
    data.push(42);
}

let mut v = vec![1, 2, 3];
process(&mut v); // explicit borrow
```

```m30
# M30 - GC handles memory
let process(data) =
    data.push(42)  # Just works

let v = [1, 2, 3]
process(v)  # No ownership concerns
```

#### 2. Structured Concurrency vs Ownership
```rust
// Rust - ownership prevents data races
let data = Arc::new(Mutex::new(vec![]));
let data_clone = Arc::clone(&data);

thread::spawn(move || {
    data_clone.lock().unwrap().push(42);
});
```

```m30
# M30 - structured concurrency + immutability
let data = []

parallel:
    let result = spawn: process_data(data)
    # Parent scope owns the lifecycle
# Automatic cleanup, no Arc/Mutex needed
```

#### 3. Fuller Type Inference
```rust
// Rust needs more annotations
let numbers: Vec<i32> = vec![1, 2, 3];
let doubled: Vec<i32> = numbers.iter()
    .map(|x| x * 2)
    .collect(); // Must specify type
```

```m30
# M30 infers more
let numbers = [1, 2, 3]
let doubled = numbers.map(x -> x * 2)  # Full inference
```

#### 4. Python-like Syntax
```m30
# Classes and methods feel like Python
class WebServer:
    mut routes: Dict[string, Handler]
    
    def add_route(self, path: string, handler: Handler) =
        self.routes[path] = handler
    
    def handle(self, request: Request) -> Response =
        match self.routes.get(request.path) with
        | Some(handler) -> handler(request)
        | None -> Response.not_found()

# List comprehensions
let evens = [x for x in numbers if x % 2 == 0]

# With statements for resources
with file = File.open("data.txt")?:
    let contents = file.read_all()?
    process(contents)
```

### Your Unique Combination

**From Python:**
- Readable syntax
- List comprehensions  
- Context managers (with)
- Import system

**From SML/OCaml:**
- Hindley-Milner inference (where possible)
- Pattern matching
- Immutability default

**From Go:**
- Fast compilation
- Built-in formatter
- Channels (but structured)
- `defer` (via `with`)

**From Rust:**
- Result/Option types
- Enums with data
- Pattern matching syntax
- Zero-cost abstractions goal

**Novel/Modern:**
- Structured concurrency (like Swift, Kotlin)
- Islands of inference
- Immutable-default classes

### Why This Combination Makes Sense

1. **Easier than Rust** - No lifetime puzzles, GC simplifies everything
2. **Safer than Python** - Static types catch errors at compile time
3. **More functional than Go** - Pattern matching, immutability, better inference
4. **More pragmatic than Haskell** - Allows mutation, familiar syntax

### Is It Worth Building?

**Yes, if you want:**
- Python's ease with static typing
- Functional features without the baggage
- Modern concurrency without async/await
- A language that compiles fast

**Similar languages to study:**
- **Gleam** - Functional, Erlang-based, similar goals
- **Roc** - Functional, fast, inference-focused
- **Nim** - Python-like syntax, compiles to C
- **Crystal** - Ruby syntax with static types

**Your niche:** Python syntax + ML inference + modern concurrency + pragmatic FP

Want to explore what makes M30 unique further? Or start defining the formal grammar?