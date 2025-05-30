# M30 Language Design v2

## Type Inference Deep Dive

### Full Hindley-Milner Inference Challenges

**What makes full inference hard:**

1. **Subtyping breaks it**:
```m30
# With subtyping, type inference becomes undecidable
class Animal
class Dog extends Animal

let f x = x  # Is this Animal -> Animal or Dog -> Dog?
```

2. **Mutable references break it**:
```ocaml
(* OCaml requires explicit types for mutable refs *)
let r = ref None  (* Error: can't infer type *)
let r : int option ref = ref None  (* OK *)
```

3. **Method overloading breaks it**:
```m30
# Can't infer which toString
let s = x.toString()  # Which overload?
```

### Proposed Solution: "Islands of Inference"

```m30
# Full inference within functions
let map f list =
    match list with
    | [] -> []
    | x :: xs -> f(x) :: map(f, xs)
# Inferred: [T, U] (T -> U, List[T]) -> List[U]

# But require annotations at module boundaries
module utils:
    # Public functions need type annotations
    export def parse_int(s: string) -> Result[int, ParseError]
    
    # Private functions can omit them
    let helper x = x * 2  # Inferred within module
```

## Concurrency Without Async/Await

### Option 1: Go-style Goroutines + Channels
```m30
# Spawn concurrent tasks that run to completion
let process_files(paths: List[string]) -> List[Result[Data, Error]] =
    let results = channel[Result[Data, Error]](len(paths))
    
    for path in paths:
        spawn:
            let result = try:
                let content = File.read(path)
                Ok(parse_data(content))
            catch e:
                Err(e)
            results.send(result)
    
    # Collect all results
    [results.recv() for _ in paths]
```

### Option 2: Erlang-style Actors
```m30
# Each actor has its own mailbox
actor Counter:
    mut count = 0
    
    receive:
        | Increment -> 
            count += 1
        | Get(reply_to) ->
            reply_to.send(count)
        | Reset ->
            count = 0

# Usage
let counter = spawn Counter()
counter ! Increment
counter ! Increment
let reply = channel[int]()
counter ! Get(reply)
print(reply.recv())  # 2
```

### Option 3: Structured Concurrency with Scopes
```m30
# All spawned tasks must complete before scope exits
let fetch_all(urls: List[string]) -> List[Data] =
    parallel scope:
        for url in urls:
            spawn: fetch(url)
    # Automatically waits for all spawns to complete
    # Returns all results in order
```

## Exceptions vs Result Types

### The Trade-offs

**Result Types (Rust/Haskell style)**:
```m30
# Explicit error handling
let divide(x: int, y: int) -> Result[float, string] =
    if y == 0:
        Err("Division by zero")
    else:
        Ok(x / y)

# Must handle errors
match divide(10, 0) with
| Ok(result) -> print(result)
| Err(msg) -> print(f"Error: {msg}")

# Chaining with ? operator
let calculate() -> Result[float, string] =
    let a = divide(10, 2)?  # Returns early if Err
    let b = divide(a, 3)?
    Ok(a + b)
```

**Exceptions (Python/Java style)**:
```m30
# Implicit error propagation
let divide(x: int, y: int) -> float =
    if y == 0:
        raise ZeroDivisionError("Division by zero")
    x / y

# Can ignore errors (dangerous!)
let result = divide(10, 0)  # Crashes if not caught

# Or handle them
try:
    let result = divide(10, 0)
catch ZeroDivisionError as e:
    print(f"Error: {e}")
```

### Proposed Hybrid Approach

```m30
# Use Result for expected errors
let parse_int(s: string) -> Result[int, ParseError] =
    # Parsing can fail, that's expected

# Use exceptions for bugs/panics
let get_item[T](list: List[T], index: int) -> T =
    if index < 0 or index >= len(list):
        panic("Index out of bounds")  # This is a bug
    list[index]

# Sugar for Result handling
let process_data(input: string) -> Result[Data, Error] =
    # ? operator for Result chaining
    let parsed = parse_int(input)?
    let validated = validate(parsed)?
    Ok(transform(validated))

# Can "catch" panics at boundaries
let safe_process(input: string) -> Result[Data, Error] =
    try:
        process_data(input)
    catch panic as p:
        Err(InternalError(p.message))
```

## Updated Syntax Examples

### Pattern Matching Everywhere
```m30
# In let bindings
let Some(user) = get_user(id) else:
    return Err("User not found")

# In for loops  
for Ok(line) in file.lines():
    process(line)

# In function parameters
let first([x, ..._]) = x
let first([]) = panic("Empty list")

# With guards
match age with
| n if n < 0 -> Err("Invalid age")
| n if n < 18 -> Ok(Minor)
| n if n < 65 -> Ok(Adult)
| _ -> Ok(Senior)
```

### Immutability by Default
```m30
class BankAccount:
    balance: float  # Immutable
    mut transactions: List[Transaction]  # Mutable
    
    # Methods can mutate mutable fields
    def deposit(self, amount: float) -> Result[float, string] =
        if amount <= 0:
            return Err("Amount must be positive")
        self.transactions.push(Deposit(amount))
        Ok(self.balance + amount)
    
    # But we return a new object for balance updates
    def with_balance(self, new_balance: float) -> BankAccount =
        BankAccount {
            balance: new_balance,
            transactions: self.transactions.copy()
        }
```

### Modules with Type Inference
```m30
module math_utils:
    # Exported functions need types
    export def sqrt(x: float) -> float
    export def abs[T: Numeric](x: T) -> T
    
    # Internal functions can use full inference
    let helper x y = x * x + y * y
    
    # Can even infer generic functions internally
    let identity x = x  # Inferred: [T] T -> T
```

## Questions

1. **Concurrency**: Which model do you prefer? Channels, actors, or structured scopes?

2. **Error handling**: Should we:
   - Use Result for all errors (pure but verbose)?
   - Hybrid approach (Result + panic)?
   - Something else?

3. **Type inference boundaries**: 
   - Require types only at module exports?
   - Also require them for class methods?
   - Or try to push full inference despite limitations?

4. **Syntax preferences**:
   - `spawn:` blocks vs `go func()` vs something else?
   - `[x, y, z]` for lists or `[x; y; z]`?
   - `mut` keyword vs `var`/`let` distinction?