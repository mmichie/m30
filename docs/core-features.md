# M30 Core Features

## Language Philosophy
**"Start exploring. Keep your work. Ship with confidence."**

M30 is designed to be the best language for exploratory programming while maintaining the safety and performance needed for production systems.

## Core Feature Set

### 1. Image-Based Development
```m30
# Save entire REPL state
> let data = load_massive_dataset()
> let model = train_model(data, epochs=100)  # Takes 2 hours
> save_image("trained-model.m30img")

# Resume later or share with colleague
$ m30 --load trained-model.m30img
> model.evaluate(test_data)  # Continue immediately
```

### 2. Multiple Dispatch
```m30
# Methods selected based on ALL argument types
def distance(p1: Point2D, p2: Point2D) -> float =
    sqrt((p1.x - p2.x)**2 + (p1.y - p2.y)**2)

def distance(p1: Point3D, p2: Point3D) -> float =
    sqrt((p1.x - p2.x)**2 + (p1.y - p2.y)**2 + (p1.z - p2.z)**2)

def distance(p: Point, line: Line) -> float =
    # Point to line distance
    abs(line.a * p.x + line.b * p.y + line.c) / sqrt(line.a**2 + line.b**2)

# Automatically selects the right method
d1 = distance(point2d_a, point2d_b)
d2 = distance(point3d, line)
```

### 3. Restartable Conditions
```m30
# Define recovery strategies at the error site
def connect_to_service(url: string) -> Connection =
    restart_case:
        socket = Socket.connect(url, timeout=5s)
        authenticate(socket)
    handle ConnectionError as e:
        | retry(new_timeout) -> 
            Socket.connect(url, timeout=new_timeout)
        | use_cached -> 
            CachedConnection(url)
        | switch_endpoint(alt_url) ->
            connect_to_service(alt_url)

# Callers choose recovery strategy
let conn = with_restarts:
    connect_to_service("https://api.example.com")
on_error:
    | ConnectionError -> retry(30s)
    | AuthError -> use_cached
```

### 4. Live Code Reloading
```m30
@reloadable
def process_request(req: Request) -> Response =
    validate(req)
    data = transform(req.data)
    Response.json(data)

# Change the function in your editor, then:
> reload(process_request)
# All future calls use new definition
# Existing data and connections remain intact!
```

### 5. Interactive Debugging
```m30
def complex_algorithm(data: List[float]) -> Analysis =
    let normalized = normalize(data)
    breakpoint()  # Drops into REPL here
    let features = extract_features(normalized)
    let result = analyze(features)
    result

# At breakpoint:
debug> normalized.mean()
0.0
debug> normalized.std()
1.0
debug> # Try different feature extraction:
debug> features = extract_features_v2(normalized)
debug> continue
# Continues with your modified data!
```

### 6. Time-Travel Debugging
```m30
# REPL maintains history
> let x = compute_something()
> let y = process(x)
> let z = transform(y)
> # Oops, wrong transform
> back 1  # Go back one step
> let z = transform_v2(y)  # Try different approach
> z
# See new result immediately
```

### 7. Compile-Time Execution
```m30
# Run code at compile time for zero-cost abstractions
const lookup_table = comptime:
    # This runs during compilation
    table = {}
    for i in 0..256:
        table[i] = expensive_computation(i)
    table

# At runtime, it's just a lookup:
def fast_compute(x: u8) -> float =
    lookup_table[x]  # O(1) lookup
```

### 8. Where Clauses
```m30
# Define helpers after the main logic
def solve_quadratic(a: float, b: float, c: float) -> Option[(float, float)] =
    if discriminant < 0 then None
    else Some((x1, x2))
  where:
    discriminant = b**2 - 4*a*c
    sqrt_disc = sqrt(discriminant)
    x1 = (-b + sqrt_disc) / (2*a)
    x2 = (-b - sqrt_disc) / (2*a)
```

### 9. Structured Concurrency
```m30
# All spawned tasks complete before scope exits
def fetch_all_data(urls: List[string]) -> Result[List[Data], Error] =
    parallel:
        for url in urls:
            spawn: fetch_and_parse(url)
    # Automatically waits for all tasks
    # Returns all results or first error
```

### 10. REPL-Aware Standard Library
```m30
# Inspection and introspection built-in
> describe(my_function)
Function: my_function
Parameters: (x: int, y: string = "default")
Returns: Option[Data]
Source: src/processing.m30:45

> trace(my_function)
# Now all calls to my_function are logged

> edit(my_function)
# Opens function in your editor

> profile:
    result = expensive_computation()
Elapsed: 1.23s
Memory: +45MB
```

## Type System Features

### Islands of Inference
```m30
module math_utils:
    # Public functions need type annotations
    export def dot_product[T: Numeric](v1: Vec[T], v2: Vec[T]) -> T
    
    # Private functions use full inference
    let helper x y = x * x + y * y  # Types inferred

# At module boundaries: explicit
# Within modules: inferred
```

### Union Types and Pattern Matching
```m30
type JsonValue = 
    | Null
    | Bool(bool)
    | Num(float)
    | Str(string)
    | Array(List[JsonValue])
    | Object(Dict[string, JsonValue])

def stringify(json: JsonValue) -> string =
    match json with
    | Null -> "null"
    | Bool(b) -> b.to_string()
    | Num(n) -> n.to_string()
    | Str(s) -> f'"{s}"'
    | Array(items) -> 
        "[" + items.map(stringify).join(", ") + "]"
    | Object(pairs) ->
        let entries = [f'"{k}": {stringify(v)}' for (k, v) in pairs]
        "{" + entries.join(", ") + "}"
```

## What Makes M30 Unique

1. **Best of Both Worlds**: Static typing with dynamic feel
2. **Exploration-First**: Image-based development for data science
3. **Production-Ready**: Compiles to native code
4. **Modern Concurrency**: Structured, safe, intuitive
5. **Lisp's Soul**: Live coding, multiple dispatch, restarts
6. **Python's Syntax**: Readable, familiar, productive

## Use Cases

- **Data Science**: Better than Jupyter notebooks
- **Systems Programming**: Safe concurrency, great performance  
- **Web Services**: Hot reload without dropping connections
- **Game Development**: Live reload assets and logic
- **Education**: Save and share entire coding sessions