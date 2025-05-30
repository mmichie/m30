# Lessons from Other Languages for M30

## Kotlin: Practical Type System Features

### 1. **Smart Casts**
```kotlin
// Kotlin automatically narrows types after checks
fun process(x: Any) {
    if (x is String) {
        println(x.length)  // x is smart-cast to String
    }
}
```

**For M30:**
```m30
let process(x: Any) =
    match x with
    | s: string -> print(s.length)  # Type narrowed in pattern
    | n: int -> print(n + 1)
```

### 2. **Extension Functions**
```kotlin
fun String.isPalindrome(): Boolean = 
    this == this.reversed()

"racecar".isPalindrome()  // true
```

**For M30:** Could allow extending types without modifying them:
```m30
extend string:
    def is_palindrome(self) -> bool =
        self == self.reverse()
```

### 3. **Sealed Classes for ADTs**
```kotlin
sealed class Result<T>
data class Success<T>(val value: T) : Result<T>()
data class Error(val message: String) : Result<Nothing>()
```

Already covered with M30's enums!

## Haskell: Functional Elegance

### 1. **Where Clauses**
```haskell
quadratic a b c = (x1, x2)
  where
    x1 = (-b + sqrt discriminant) / (2 * a)
    x2 = (-b - sqrt discriminant) / (2 * a)
    discriminant = b^2 - 4*a*c
```

**For M30:** Bottom-up definitions can improve readability:
```m30
let quadratic a b c = (x1, x2)
  where:
    discriminant = b**2 - 4*a*c
    x1 = (-b + sqrt(discriminant)) / (2 * a)
    x2 = (-b - sqrt(discriminant)) / (2 * a)
```

### 2. **Operators as Functions**
```haskell
map (*2) [1,2,3]  -- [2,4,6]
(+) 3 4           -- 7
(< 5)             -- partial application
```

**For M30:**
```m30
let doubles = map((*2), [1, 2, 3])
let add = (+)  # Operators as first-class functions
let less_than_5 = (< 5)  # Partial application
```

### 3. **List Comprehensions with Multiple Generators**
```haskell
[(x,y) | x <- [1..3], y <- [1..3], x < y]
-- [(1,2),(1,3),(2,3)]
```

**For M30:**
```m30
[(x, y) for x in 1..3 for y in 1..3 if x < y]
```

## Lisp: Metaprogramming Power

### 1. **Condition System (Better than Exceptions)**
```lisp
(define-condition file-not-found-error (error) ...)

(restart-case 
    (open-file filename)
  (use-default ()
    :report "Use default config"
    (open-file "default.conf"))
  (retry ()
    :report "Try again"
    (open-file filename)))
```

**For M30:** Restartable conditions for better error recovery:
```m30
let open_config(filename) =
    restart_case:
        File.open(filename)
    with:
        | use_default -> File.open("default.conf")
        | create_new -> File.create(filename)
        | retry -> open_config(filename)
```

### 2. **Multiple Dispatch**
```lisp
(defmethod collide ((x asteroid) (y spaceship)) ...)
(defmethod collide ((x spaceship) (y spaceship)) ...)
```

**For M30:** Method dispatch on multiple arguments:
```m30
def collide(a: Asteroid, s: Spaceship) = ...
def collide(s1: Spaceship, s2: Spaceship) = ...
```

## Erlang/Elixir: Fault Tolerance

### 1. **Supervision Trees**
```elixir
defmodule MyApp.Supervisor do
  use Supervisor
  
  def start_link(opts) do
    children = [
      {Worker, arg},
      {AnotherWorker, arg}
    ]
    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

**For M30:** Structured concurrency with restart strategies:
```m30
let supervised[T](restart_strategy, tasks) -> Result[List[T], Error] =
    parallel supervised(restart_strategy):
        for task in tasks:
            spawn: task()
    # Automatically restarts failed tasks based on strategy
```

### 2. **Pattern Matching on Messages**
```elixir
receive do
  {:hello, name} -> "Hello #{name}"
  {:goodbye, name} -> "Bye #{name}"
  _ -> "Unknown message"
after
  5000 -> "Timeout"
end
```

**For M30:** Actor-style message passing:
```m30
actor Counter:
    mut state = 0
    
    receive:
        | Increment -> state += 1
        | Decrement -> state -= 1
        | Get(reply_to) -> reply_to.send(state)
    timeout 5s:
        panic("No messages received")
```

## Prolog: Logic Programming

### 1. **Unification in Pattern Matching**
```prolog
append([], L, L).
append([H|T1], L2, [H|T3]) :- append(T1, L2, T3).
```

**For M30:** Bidirectional pattern matching:
```m30
# Pattern matching can flow backwards
match (x, [1, 2, 3]) with
| ([], ys) -> ys
| ([h, ...t], ys) -> [h, ...append(t, ys)]

# Can "run backwards" to solve for x:
# append(x, [3], [1, 2, 3]) => x = [1, 2]
```

## Experimental Language Features

### 1. **Zig's Comptime**
```zig
fn fibonacci(comptime n: u32) u32 {
    comptime var a = 0;
    comptime var b = 1;
    comptime var i = 0;
    inline while (i < n) : (i += 1) {
        const tmp = a + b;
        a = b;
        b = tmp;
    }
    return a;
}
```

**For M30:** Compile-time evaluation:
```m30
const fibonacci[n: comptime int] -> int =
    comptime:
        # This runs at compile time
        let mut a = 0
        let mut b = 1
        for _ in 0..n:
            (a, b) = (b, a + b)
        a

let x = fibonacci[10]  # Computed at compile time
```

### 2. **Roc's Abilities (Like Rust Traits but Better)**
```roc
Hash has
    hash : a -> U64 | a has Hash

Eq has
    equals : a, a -> Bool | a has Eq
```

**For M30:** Structural interfaces:
```m30
ability Hash[T]:
    hash(T) -> int

ability Eq[T]:
    equals(T, T) -> bool

# Types automatically implement if they have the methods
type Point = { x: float, y: float }

# Automatic implementation based on structure
def hash(p: Point) -> int =
    hash(p.x) ^ hash(p.y)
```

### 3. **Unison's Content-Addressed Code**
```unison
-- Functions identified by hash, not name
> hash myFunction
#a8f7sd9f7s9d7f
```

**For M30:** Could enable perfect caching and distributed execution:
```m30
# Functions are pure and content-addressed
@pure
@cached
let expensive_compute(x) = ...

# Same function with same input always returns cached result
# Can be computed on any machine in a cluster
```

## The Most Important Lessons

1. **Kotlin's Null Safety** - Make `None` handling explicit in type system
2. **Haskell's Where Clauses** - Sometimes bottom-up is clearer
3. **Lisp's Restarts** - Better error recovery than just catch
4. **Erlang's Supervision** - Self-healing concurrent systems
5. **Prolog's Unification** - Patterns could be bidirectional
6. **Zig's Comptime** - Compile-time execution for zero-cost abstractions
7. **Smalltalk's Image** - REPL could maintain state between sessions

## What M30 Could Uniquely Combine

1. **Gradual Logic Programming**
```m30
# Normal function
let append(xs, ys) = xs + ys

# Logic mode - can run backwards
logic append([], L, L).
logic append([H|T1], L2, [H|T3]) :- append(T1, L2, T3).

# Query: append(X, [3], [1,2,3]) => X = [1,2]
```

2. **Effect Tracking Without Monads**
```m30
# Effects tracked in type system but no monad syntax
def read_file(path: string) -> string ! {IO, FileError} =
    File.read(path)

# Compiler ensures effects are handled
def pure_function(x: int) -> int ! {} =  # No effects allowed
    x * 2
```

3. **Time-Travel Debugging in REPL**
```m30
# REPL saves all states
> let x = 5
> let y = x * 2
> let z = y + 3
> back 2  # Go back 2 steps
> let y = x * 3  # Change history
> forward
> z  # Now 18 instead of 13
```

Which of these features excite you most? Any seem essential for M30's vision?