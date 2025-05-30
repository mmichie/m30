# M30 Design v3: Concrete Examples

## Structured Concurrency Deep Dive

### Why Structured Concurrency?

Traditional concurrency (threads/goroutines) has problems:
- Tasks can outlive their parent
- No automatic error propagation
- Resource cleanup is manual
- Hard to reason about lifetime

Structured concurrency guarantees:
- All child tasks complete before parent continues
- Errors automatically propagate
- Resources cleaned up in reverse order
- Clear parent-child relationships

### Basic Examples

```m30
# Sequential version
let fetch_user_data(id: int) -> Result[UserData, Error] =
    let profile = fetch_profile(id)?
    let posts = fetch_posts(id)?
    let friends = fetch_friends(id)?
    Ok(UserData { profile, posts, friends })

# Parallel version with structured concurrency
let fetch_user_data(id: int) -> Result[UserData, Error] =
    parallel:
        let profile = spawn: fetch_profile(id)
        let posts = spawn: fetch_posts(id)
        let friends = spawn: fetch_friends(id)
    # Waits for all spawns to complete
    # If any return Err, the first error is propagated
    Ok(UserData { profile?, posts?, friends? })
```

### Scope-based Resource Management

```m30
# Resources are automatically cleaned up
let process_file(path: string) -> Result[Data, Error] =
    with file = File.open(path)?:
        parallel:
            let header = spawn: parse_header(file)?
            let body = spawn: parse_body(file)?
        Ok(Data { header?, body? })
    # File automatically closed here, even if error occurs
```

### Cancellation and Timeouts

```m30
# All child tasks cancelled if parent scope exits early
let fetch_with_timeout[T](url: string, timeout: Duration) -> Result[T, Error] =
    race:
        let data = spawn: fetch(url)
        let _ = spawn: sleep(timeout)
    # First to complete wins, other is cancelled
    match data:
        | Some(result) -> result
        | None -> Err(TimeoutError)

# Manual cancellation
let search_files(pattern: string, dirs: List[string]) -> Result[File, Error] =
    parallel:
        for dir in dirs:
            spawn:
                for file in walk_dir(dir):
                    if file.matches(pattern):
                        return Ok(file)  # Cancels all other searches
    Err(NotFound)
```

### Error Handling in Concurrent Code

```m30
# Collecting all results (even errors)
let batch_process(items: List[Item]) -> List[Result[Output, Error]] =
    parallel:
        for item in items:
            spawn: process_item(item)
    # Returns list of all results in order

# Fail fast on first error
let validate_all(items: List[Item]) -> Result[List[Valid], Error] =
    parallel:
        for item in items:
            spawn: validate(item)
    # If any validation fails, returns first error immediately
    # All other validations are cancelled
```

### Comparison with Go-style Channels

```m30
# Go-style (what we're NOT doing)
let process_go_style(items: List[Item]) -> List[Result[Output, Error]] =
    let ch = channel[Result[Output, Error]](len(items))
    
    for item in items:
        go:  # Fire and forget!
            ch.send(process_item(item))
    
    # Must manually collect results
    let results = []
    for _ in items:
        results.push(ch.recv())
    results
    # Problems: 
    # - Goroutines might outlive function
    # - No automatic cancellation
    # - Manual coordination

# M30 structured style
let process_m30_style(items: List[Item]) -> List[Result[Output, Error]] =
    parallel:
        for item in items:
            spawn: process_item(item)
    # Benefits:
    # - All tasks complete before function returns
    # - Automatic result collection
    # - Automatic cancellation on error
```

## Complete Example: Web Scraper

```m30
module web_scraper:
    # Module exports require type annotations
    export def scrape_site(url: string, max_depth: int) -> Result[SiteData, Error]
    
    type Link = { url: string, text: string }
    type Page = { url: string, title: string, links: List[Link] }
    type SiteData = { pages: List[Page], total_links: int }
    
    # Implementation with full inference internally
    let scrape_site(url: string, max_depth: int) -> Result[SiteData, Error] =
        let visited = HashSet.new[string]()
        let pages = scrape_recursive(url, max_depth, visited)?
        let total_links = pages.flat_map(p -> p.links).length()
        Ok(SiteData { pages, total_links })
    
    # Internal functions use inference
    let scrape_recursive(url, depth, visited) =
        if depth == 0 or visited.contains(url):
            return Ok([])
        
        visited.insert(url)
        let page = fetch_page(url)?
        
        # Parallel recursive scraping
        parallel:
            let child_pages = for link in page.links:
                if should_follow(link):
                    spawn: scrape_recursive(link.url, depth - 1, visited)
                else:
                    Ok([])
        
        # Collect all results, filtering out errors
        let all_pages = child_pages
            .filter_map(r -> r.ok())
            .flatten()
        
        Ok([page] + all_pages)
    
    # Pattern matching for HTML parsing
    let extract_links(html) =
        html.find_all("a")
            .filter_map(tag ->
                match (tag.get("href"), tag.text()) with
                | (Some(url), text) if url.starts_with("http") ->
                    Some(Link { url, text })
                | _ -> None
            )
```

## Islands of Inference in Practice

```m30
# math_lib.m30
module math_lib:
    # Public API needs types
    export def dot_product[T: Numeric](v1: Vec[T], v2: Vec[T]) -> Result[T, Error]
    export def normalize[T: Float](v: Vec[T]) -> Vec[T]
    
    # But internally, full inference
    let dot_product(v1, v2) =
        if v1.len() != v2.len():
            return Err("Vectors must have same length")
        
        Ok(sum(zip(v1, v2).map(|(a, b)| a * b)))
    
    # Generic functions inferred
    let magnitude(v) = 
        sqrt(dot_product(v, v).unwrap())
    
    let normalize(v) =
        let mag = magnitude(v)
        v.map(x -> x / mag)

# Using the module - types are known
import math_lib

let v1: Vec[float] = [1.0, 2.0, 3.0]
let v2 = [4.0, 5.0, 6.0]  # Type inferred from context
let result = math_lib.dot_product(v1, v2)?  # Returns Result[float, Error]
```

## Design Questions Resolved

1. **Error Propagation in Parallel Blocks**: 
   - First error cancels all other tasks and propagates
   - Use `parallel collect:` to get all results (including errors)

2. **Resource Management**:
   - `with` blocks for RAII-style cleanup
   - Works correctly with parallel blocks

3. **Type Inference Boundary**:
   - Module exports (public functions)
   - Class public methods
   - Everything else inferred

Would you like me to explore any specific aspect deeper? Maybe:
- How parallel blocks interact with mutable state?
- More complex pattern matching examples?
- Module system and visibility rules?