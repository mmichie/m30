# M30 Language Design Exploration

## Key Concepts Explained

### Expression vs Statement Distinction

**Statements** don't return values:
```python
# Python statements
x = 5          # assignment statement
if x > 3:      # if statement
    print(x)   # print statement
```

**Expressions** always return values:
```rust
// Rust - everything is an expression
let x = if y > 3 { 5 } else { 10 };  // if returns a value
let z = {
    let a = 5;
    a + 10     // block returns last expression
}; // z = 15
```

### Hindley-Milner Type Inference

Full HM inference means the compiler figures out ALL types:
```ml
(* OCaml - no type annotations needed *)
let rec map f = function
  | [] -> []
  | x::xs -> f x :: map f xs
(* Compiler infers: ('a -> 'b) -> 'a list -> 'b list *)
```

**Difficulty**: HM inference becomes undecidable with:
- Subtyping (OOP inheritance)
- Mutable references
- Higher-rank polymorphism

Most languages compromise (Rust, Swift, Kotlin require some annotations).

### OOP vs Functional Impedance Mismatch

**OOP** uses nominal typing (types by name):
```python
class Dog:
    def bark(self): ...

class Cat:
    def bark(self): ...
    
# Dog and Cat are different types even with same interface
```

**Functional** often uses structural typing:
```typescript
// TypeScript structural typing
type Barker = { bark: () => void }
// Any object with bark() method works
```

## Modern Concurrency Models

1. **Go**: Goroutines + channels (CSP model)
2. **Rust**: Ownership prevents data races, async/await
3. **Erlang/Elixir**: Actor model (isolated processes)
4. **Swift**: Structured concurrency, async/await with actors

## Pattern Matching Examples

```rust
// Rust pattern matching
match value {
    Some(x) if x > 0 => x * 2,
    Some(0) => 0,
    None => -1,
}
```

```ml
(* OCaml *)
match tree with
| Leaf n -> n
| Node(left, right) -> sum left + sum right
```

---

# M30 Language Design Proposal v1

## Core Principles
- Readability first (Python-like)
- Static typing with inference
- Functional-first but pragmatic
- Safe concurrency
- Fast compilation

## Syntax Examples

### Basic Syntax
```m30
# Variable declaration with optional type
let x = 5
let y: int = 10

# Type inference
let add a b = a + b  # inferred: (num, num) -> num

# With type annotations
let add (a: int) (b: int) -> int =
    a + b

# Multiline functions use indentation
let factorial n =
    if n <= 1 then
        1
    else
        n * factorial (n - 1)
```

### Data Types and Pattern Matching
```m30
# Enums (algebraic data types)
type Option[T] =
    | Some(T)
    | None

type Result[T, E] =
    | Ok(T)
    | Err(E)

# Pattern matching
let divide x y =
    match y with
    | 0 -> Err("Division by zero")
    | _ -> Ok(x / y)

# Pattern matching in let
let Some(value) = might_be_none else
    return Err("was none")
```

### Classes and OOP
```m30
# Classes with immutable fields by default
class Point:
    x: float
    y: float
    
    # Constructor
    new(x, y) =
        Point { x, y }
    
    # Methods
    def distance_to(other: Point) -> float =
        let dx = self.x - other.x
        let dy = self.y - other.y
        sqrt(dx * dx + dy * dy)

# Inheritance
class Point3D extends Point:
    z: float
    
    new(x, y, z) =
        Point3D { x, y, z }
    
    # Override with pattern matching
    override def distance_to(other: Point) -> float =
        match other with
        | Point3D { x, y, z } ->
            let dx = self.x - x
            let dy = self.y - y
            let dz = self.z - z
            sqrt(dx * dx + dy * dy + dz * dz)
        | Point { x, y } ->
            super.distance_to(other)
```

### Generics and Constraints
```m30
# Rust-style generics
let identity[T](x: T) -> T = x

# With constraints
let print_all[T: Display](items: List[T]) =
    for item in items:
        print(item)

# Multiple constraints
let process[T: Hash + Eq](item: T) =
    # ...
```

### Concurrency
```m30
# Async/await with structured concurrency
async def fetch_data(url: string) -> Result[Data, Error] =
    let response = await http.get(url)
    match response.status with
    | 200 -> Ok(response.parse[Data]())
    | code -> Err(f"HTTP {code}")

# Parallel execution
async def fetch_all(urls: List[string]) -> List[Result[Data, Error]] =
    await.all(urls.map(fetch_data))

# Channels for communication
let (sender, receiver) = channel[int]()

# Spawn concurrent task
spawn:
    for i in range(10):
        sender.send(i)
    sender.close()

# Receive values
while let Some(value) = receiver.recv():
    print(value)
```

### Error Handling
```m30
# Exceptions for truly exceptional cases
def read_config(path: string) -> Config =
    try:
        let data = File.read(path)?  # ? propagates Result errors
        parse_config(data)
    catch FileNotFound:
        Config.default()
    catch ParseError as e:
        raise ConfigError(f"Invalid config: {e}")

# Result type for expected errors
def parse_int(s: string) -> Result[int, ParseError] =
    # implementation
```

### Modules and Imports
```m30
# Python-style imports
import std.collections (HashMap, HashSet)
import math
from mylib.utils import helper as h

# Module definition
module geometry:
    export Point, distance
    
    class Point:
        # ...
    
    let distance p1 p2 =
        # ...
```

## Type System Features

1. **Gradual inference** - infer what we can, annotate boundaries
2. **Union types** - `int | string`
3. **Intersection types** - `Readable & Writable`
4. **Type aliases** - `type UserId = int`
5. **Const generics** - `Array[T, 10]`

## Questions for Iteration

1. **Function syntax**: Do you prefer `let add a b =` or `def add(a, b):`?
2. **Type annotation style**: `: int` or `-> int` for return types?
3. **Pipeline operator**: `data |> filter(x -> x > 0) |> map(x -> x * 2)`?
4. **String interpolation**: f-strings `f"{name}"` or something else?
5. **Async syntax**: `async/await` or something more integrated?

What do you think? What would you change?