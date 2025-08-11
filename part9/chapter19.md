# Chapter 19: Experimental Features

## The Future Being Written

Go's evolution is careful and deliberate. Features spend years in discussion, experimentation, and refinement before joining the language. This chapter explores experimental features currently being developed, proposed enhancements under consideration, and the glimpses we have of Go's future. While these features may change or never ship, understanding them helps us see where Go is heading.

## Range Over Functions (Go 1.23+)

One of the most significant experimental features is the ability to range over functions, enabling custom iterators:

```go
package main

import (
    "iter"
    "fmt"
)

// Range over functions allows custom iteration
func CountTo(n int) iter.Seq[int] {
    return func(yield func(int) bool) {
        for i := 1; i <= n; i++ {
            if !yield(i) {
                return
            }
        }
    }
}

// Fibonacci iterator
func Fibonacci(max int) iter.Seq[int] {
    return func(yield func(int) bool) {
        a, b := 0, 1
        for a <= max {
            if !yield(a) {
                return
            }
            a, b = b, a+b
        }
    }
}

// Tree iterator
type Node[T any] struct {
    Value T
    Left  *Node[T]
    Right *Node[T]
}

func (n *Node[T]) InOrder() iter.Seq[T] {
    return func(yield func(T) bool) {
        var traverse func(*Node[T]) bool
        traverse = func(node *Node[T]) bool {
            if node == nil {
                return true
            }
            
            if !traverse(node.Left) {
                return false
            }
            
            if !yield(node.Value) {
                return false
            }
            
            return traverse(node.Right)
        }
        
        traverse(n)
    }
}

// Two-value iterators
func Enumerate[T any](slice []T) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        for i, v := range slice {
            if !yield(i, v) {
                return
            }
        }
    }
}

func main() {
    // Use custom iterators with for-range
    for n := range CountTo(5) {
        fmt.Println(n)  // 1, 2, 3, 4, 5
    }
    
    for fib := range Fibonacci(100) {
        fmt.Println(fib)  // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89
    }
    
    // Early termination works
    for n := range CountTo(1000) {
        if n > 10 {
            break  // Iterator stops cleanly
        }
        fmt.Println(n)
    }
    
    // Two-value iteration
    data := []string{"a", "b", "c"}
    for i, v := range Enumerate(data) {
        fmt.Printf("%d: %s\n", i, v)
    }
}

// Advanced: Pull iterators
func PullExample() {
    // Convert push-style to pull-style
    next, stop := iter.Pull(CountTo(10))
    defer stop()
    
    // Pull values manually
    for {
        v, ok := next()
        if !ok {
            break
        }
        fmt.Println(v)
    }
}

// Composable iterators
func Filter[T any](seq iter.Seq[T], predicate func(T) bool) iter.Seq[T] {
    return func(yield func(T) bool) {
        for v := range seq {
            if predicate(v) {
                if !yield(v) {
                    return
                }
            }
        }
    }
}

func Map[T, U any](seq iter.Seq[T], f func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range seq {
            if !yield(f(v)) {
                return
            }
        }
    }
}

func Take[T any](seq iter.Seq[T], n int) iter.Seq[T] {
    return func(yield func(T) bool) {
        count := 0
        for v := range seq {
            if count >= n {
                return
            }
            if !yield(v) {
                return
            }
            count++
        }
    }
}

// Usage with composition
func ComposedExample() {
    numbers := CountTo(100)
    evens := Filter(numbers, func(n int) bool { return n%2 == 0 })
    squared := Map(evens, func(n int) int { return n * n })
    firstTen := Take(squared, 10)
    
    for n := range firstTen {
        fmt.Println(n)  // 4, 16, 36, 64, 100, 144, 196, 256, 324, 400
    }
}
```

## Generic Type Aliases (Proposed)

Generic type aliases would allow creating aliases for generic types:

```go
// Proposed syntax (not yet available)
type StringMap[V any] = map[string]V
type Result[T any] = struct {
    Value T
    Error error
}

// Would allow
var users StringMap[User]
var result Result[string]

// Instead of current workarounds
type StringMap[V any] map[string]V  // Not an alias, new type
```

## Structured Logging Improvements

Future enhancements to log/slog are being explored:

```go
// Proposed: Context-aware logging
func ProposedContextLogging(ctx context.Context) {
    // Automatic context extraction
    slog.InfoContext(ctx, "processing request")
    // Would automatically include trace_id, span_id, etc.
    
    // Proposed: Sampling
    logger := slog.New(slog.NewSamplingHandler(
        slog.NewJSONHandler(os.Stdout, nil),
        slog.SamplingOptions{
            Level: slog.LevelDebug,
            Rate:  0.1,  // Sample 10% of debug logs
        },
    ))
    
    // Proposed: Async logging
    logger = slog.New(slog.NewAsyncHandler(
        slog.NewJSONHandler(os.Stdout, nil),
        slog.AsyncOptions{
            BufferSize: 10000,
            Workers:    4,
        },
    ))
}
```

## Arena Memory Management (Experimental)

Arena allocation for reducing GC pressure:

```go
// Experimental arena package
import "arena"

func ArenaExample() {
    // Create arena
    a := arena.New()
    defer a.Free()
    
    // Allocate in arena
    ptr := arena.New[MyStruct](a)
    slice := arena.MakeSlice[int](a, 0, 100)
    
    // All memory freed at once when arena is freed
    // Reduces GC pressure for temporary allocations
}

// Proposed use cases
func ProcessLargeDataSet(data []byte) Result {
    a := arena.New()
    defer a.Free()
    
    // All intermediate allocations use arena
    parsed := arena.New[ParsedData](a)
    parsed.Parse(data)
    
    intermediate := arena.MakeSlice[IntermediateResult](a, 0, 1000)
    // Process...
    
    // Only final result escapes arena
    return computeResult(intermediate)
}
```

## Telemetry in Go Toolchain

Proposed telemetry for improving Go:

```go
// Proposed telemetry configuration
// go.telemetry file
telemetry:
  enabled: true
  upload: weekly
  counters:
    - go/build
    - go/test
    - go/packages
  
// Opt-in telemetry for better tooling
// $ go telemetry on
// $ go telemetry view
```

## Sum Types / Discriminated Unions (Proposed)

One of the most requested features:

```go
// Proposed syntax variations

// Option 1: Union types
type Result union {
    Success struct {
        Value string
    }
    Error struct {
        Code    int
        Message string
    }
}

// Option 2: Sealed interfaces
type Result sealed interface {
    case Success(value string)
    case Error(code int, message string)
}

// Usage with type switch
func handle(r Result) {
    switch v := r.(type) {
    case Success:
        fmt.Println("Success:", v.Value)
    case Error:
        fmt.Printf("Error %d: %s\n", v.Code, v.Message)
    // Compiler knows all cases are covered
    }
}

// Current workaround
type Result struct {
    Success *SuccessResult
    Error   *ErrorResult
}

func (r Result) IsSuccess() bool { return r.Success != nil }
func (r Result) IsError() bool { return r.Error != nil }
```

## Improved Error Handling (Proposals)

Various proposals for error handling:

```go
// Proposal 1: try keyword
func ProposedTry() (string, error) {
    data := try(os.ReadFile("file.txt"))
    parsed := try(parseData(data))
    result := try(process(parsed))
    return result, nil
}

// Proposal 2: Error handling expressions
func ProposedErrorExpr() (string, error) {
    data := os.ReadFile("file.txt") ?? return "", err
    parsed := parseData(data) ?? return "", err
    return process(parsed)
}

// Proposal 3: Check/handle blocks
func ProposedCheckHandle() (string, error) {
    check {
        data := os.ReadFile("file.txt")
        parsed := parseData(data)
        return process(parsed), nil
    } handle err {
        return "", fmt.Errorf("processing failed: %w", err)
    }
}

// Current reality: explicit is here to stay
func CurrentReality() (string, error) {
    data, err := os.ReadFile("file.txt")
    if err != nil {
        return "", err
    }
    
    parsed, err := parseData(data)
    if err != nil {
        return "", err
    }
    
    return process(parsed)
}
```

## Coroutines (Far Future)

Exploring coroutines for different concurrency models:

```go
// Hypothetical coroutine support
func CoroutineExample() {
    // Stackless coroutines
    coro := coroutine.New(func(yield func(int)) {
        for i := 0; i < 10; i++ {
            yield(i)
        }
    })
    
    for coro.Resume() {
        fmt.Println(coro.Value())
    }
    
    // Async/await style (very hypothetical)
    async func fetchData() ([]byte, error) {
        result := await http.Get("https://api.example.com")
        return await io.ReadAll(result.Body)
    }
}
```

## Language Server Protocol Enhancements

Future gopls capabilities:

```go
// Proposed: AI-assisted coding
// gopls could provide:
// - Intelligent code completion
// - Automatic refactoring suggestions
// - Performance optimization hints
// - Security vulnerability detection

// Proposed: Advanced refactoring
// - Extract interface from struct
// - Convert between patterns
// - Automatic error wrapping
// - Generate tests from examples
```

## WebAssembly System Interface (WASI)

Improved WebAssembly support:

```go
//go:build wasi

package main

import (
    "os"
    "fmt"
)

// Future WASI support would allow
func WASIExample() {
    // File system access
    file, err := os.Open("/data/input.txt")
    
    // Network access (proposed)
    conn, err := net.Dial("tcp", "example.com:80")
    
    // Threads (proposed)
    go func() {
        // Goroutines in WASM
    }()
}

// Compile for WASI
// $ GOOS=wasip1 GOARCH=wasm go build -o app.wasm
// $ wasmtime app.wasm
```

## Contracts (Alternative to Constraints)

Earlier generic proposals included contracts:

```go
// Original contracts proposal (rejected in favor of type sets)
contract Ordered(T) {
    T < T
    T <= T
    T > T
    T >= T
}

func Min[T Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// What we got instead (type sets)
type Ordered interface {
    ~int | ~float64 | ~string
}
```

## Optimization Directives

Proposed compiler hints:

```go
// Proposed optimization hints

//go:hot
func criticalPath() {
    // Optimize aggressively
}

//go:cold
func errorHandler() {
    // Rarely executed
}

//go:branchprediction likely
if condition {
    // Likely branch
}

//go:vectorize
func compute(a, b []float64) []float64 {
    // Enable auto-vectorization
}

//go:unroll 4
for i := 0; i < len(data); i++ {
    // Unroll loop 4 times
}
```

## Module Workspaces Evolution

Future workspace enhancements:

```go
// Proposed: Workspace dependencies
// go.work
use (
    ./service-a
    ./service-b
    ./shared
)

require (
    // Workspace-level dependencies
    github.com/stretchr/testify v1.8.4
)

replace (
    // Workspace-level replacements
    github.com/example/lib => ./lib
)

// Proposed: Workspace commands
// $ go work sync    # Sync all modules
// $ go work test    # Test all modules
// $ go work build   # Build all modules
```

## Type Parameter Inference Improvements

Enhanced type inference:

```go
// Current limitation
func Ptr[T any](v T) *T {
    return &v
}

// Must specify type sometimes
var p *int = Ptr[int](42)

// Proposed: Better inference
var p *int = Ptr(42)  // T inferred as int

// Proposed: Inference from return type
func Zero[T any]() T {
    var zero T
    return zero
}

var x int = Zero()  // T inferred as int
```

## Native JSON Support

Proposals for better JSON handling:

```go
// Proposed: JSON literals
data := json`{
    "name": "Alice",
    "age": 30,
    "emails": ["alice@example.com"]
}`

// Proposed: JSON pattern matching
switch json {
case `{"type": "user", "id": ?id}`:
    handleUser(id)
case `{"type": "admin", ...rest}`:
    handleAdmin(rest)
}

// Proposed: Compile-time JSON validation
type User struct {
    Name string `json:"name" jsonschema:"required,minLength=1"`
    Age  int    `json:"age" jsonschema:"minimum=0,maximum=150"`
}
```

## Forward Compatibility

Go's approach to new features:

```go
// GODEBUG for gradual adoption
// GODEBUG=gotypesalias=1 go build  // Enable type aliases

// Build tags for experimental features
//go:build goexperiment.arenas

// Version-specific behavior
//go:build go1.24
```

## Community Proposals

Popular community requests:

```go
// Ternary operator (repeatedly rejected)
result := condition ? valueTrue : valueFalse

// Null coalescing (under discussion)
value := maybeNil ?? defaultValue

// Optional chaining (proposed)
city := user?.address?.city

// Pattern matching (various proposals)
match value {
    case 0..10: handleSmall()
    case 11..100: handleMedium()
    case _: handleLarge()
}

// Operator overloading (likely never)
func (v Vector) operator+(other Vector) Vector {
    // Custom + operator
}
```

## The Go 2 Discussion

Long-term vision discussions:

```go
// Go 2 goals (not a breaking change)
// - Improved error handling
// - Better generics
// - Enhanced tooling
// - Simplified syntax where possible

// Compatibility promise
// Go 1 code will continue to work
```

## Best Practices for Experimental Features

### 1. Stay Informed
```bash
# Follow proposals
# https://github.com/golang/go/issues

# Read design documents
# https://go.googlesource.com/proposal
```

### 2. Test in Isolation
```go
//go:build experimental

// Keep experimental code separate
```

### 3. Provide Feedback
```go
// Test proposed features
// Report issues and use cases
// Participate in discussions
```

### 4. Plan for Change
```go
// Experimental features may:
// - Change significantly
// - Be removed entirely
// - Take years to stabilize
```

## Exercises

1. **Custom Iterator**: Implement a complex iterator using range-over-func.

2. **Arena Allocator**: Simulate arena allocation patterns with current Go.

3. **Error Handling Pattern**: Design your ideal error handling pattern.

4. **Type System Extension**: Propose a type system feature you'd like to see.

5. **Tooling Enhancement**: Create a prototype of a gopls feature you want.

## Summary

Go's experimental features show a language evolving thoughtfully. From iterators to improved error handling, from arena allocation to sum types, these proposals address real pain points while maintaining Go's simplicity. Not all will make it into the language, but each contributes to the discussion of Go's future.

Key takeaways:
- Range-over-functions enables powerful iteration patterns
- Memory management improvements are being explored
- Error handling remains an active area of discussion
- Type system enhancements are carefully considered
- The community actively shapes Go's evolution

Next, we'll explore the broader Go ecosystem and its vibrant community.

---

*Continue to [Chapter 20: The Go Ecosystem](chapter20.md)*