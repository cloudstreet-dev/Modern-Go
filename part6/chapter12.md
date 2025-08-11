# Chapter 12: New Standard Packages

## The Expanding Universe

Go's standard library philosophy is conservative: new packages must solve real problems, have proven designs, and commit to long-term stability. Since 2015, each addition has been battle-tested in the ecosystem before promotion. This chapter explores the packages that made the cut—and why they matter.

## log/slog: Structured Logging (Go 1.21+)

Finally, Go has structured logging in the standard library:

```go
package main

import (
    "context"
    "log/slog"
    "os"
    "time"
)

func basicSlog() {
    // Default logger (text to stderr)
    slog.Info("server started", 
        "port", 8080,
        "env", "production")
    
    // With attributes
    slog.Error("database connection failed",
        "host", "db.example.com",
        "error", "connection refused",
        "retry_in", 5*time.Second)
    
    // Grouped attributes
    slog.Info("request completed",
        slog.Group("request",
            "method", "GET",
            "path", "/api/users",
            "duration", 125*time.Millisecond),
        slog.Group("response",
            "status", 200,
            "bytes", 2048))
}

// Custom handlers
func customHandlers() {
    // JSON handler for production
    jsonHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
        AddSource: true,
    })
    jsonLogger := slog.New(jsonHandler)
    jsonLogger.Info("structured output", "format", "json")
    // Output: {"time":"2024-01-15T10:30:00Z","level":"INFO","source":{"function":"main.customHandlers","file":"/app/main.go","line":42},"msg":"structured output","format":"json"}
    
    // Text handler for development
    textHandler := slog.NewTextHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelDebug,
        ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
            // Customize attribute output
            if a.Key == slog.TimeKey {
                return slog.Attr{
                    Key:   "timestamp",
                    Value: slog.StringValue(time.Now().Format(time.RFC3339)),
                }
            }
            return a
        },
    })
    textLogger := slog.New(textHandler)
    textLogger.Debug("detailed output", "format", "text")
}

// Context integration
func contextLogging() {
    // Add logger to context
    ctx := context.Background()
    logger := slog.Default().With(
        "request_id", "abc-123",
        "user_id", "user-456",
    )
    
    ctx = context.WithValue(ctx, "logger", logger)
    
    // Use logger from context
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    logger := ctx.Value("logger").(*slog.Logger)
    logger.Info("processing request")
    
    // Add more context
    logger = logger.With("step", "validation")
    logger.Info("validating input")
}

// Custom logger implementation
type CustomHandler struct {
    level slog.Level
    attrs []slog.Attr
    groups []string
}

func (h *CustomHandler) Enabled(ctx context.Context, level slog.Level) bool {
    return level >= h.level
}

func (h *CustomHandler) Handle(ctx context.Context, r slog.Record) error {
    // Custom handling logic
    fmt.Printf("[%s] %s", r.Level, r.Message)
    
    r.Attrs(func(a slog.Attr) bool {
        fmt.Printf(" %s=%v", a.Key, a.Value)
        return true
    })
    
    fmt.Println()
    return nil
}

func (h *CustomHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
    return &CustomHandler{
        level:  h.level,
        attrs:  append(h.attrs, attrs...),
        groups: h.groups,
    }
}

func (h *CustomHandler) WithGroup(name string) slog.Handler {
    return &CustomHandler{
        level:  h.level,
        attrs:  h.attrs,
        groups: append(h.groups, name),
    }
}

// Performance-conscious logging
func performantLogging() {
    logger := slog.Default()
    
    // Avoid allocations with LogAttrs
    logger.LogAttrs(context.Background(), slog.LevelInfo, "efficient logging",
        slog.Int("count", 42),
        slog.String("status", "ok"),
        slog.Duration("elapsed", 100*time.Millisecond))
    
    // Conditional expensive operations
    if logger.Enabled(context.Background(), slog.LevelDebug) {
        expensive := computeExpensiveDebugInfo()
        logger.Debug("detailed info", "data", expensive)
    }
}
```

## slices Package: Generic Slice Operations

The slices package provides efficient, generic operations:

```go
package main

import (
    "cmp"
    "fmt"
    "slices"
)

func slicesOperations() {
    nums := []int{3, 1, 4, 1, 5, 9, 2, 6}
    
    // Sorting
    slices.Sort(nums)
    fmt.Println(nums)  // [1 1 2 3 4 5 6 9]
    
    // Binary search (on sorted slice)
    index, found := slices.BinarySearch(nums, 5)
    fmt.Printf("5 found at index %d: %v\n", index, found)
    
    // Check if sorted
    if slices.IsSorted(nums) {
        fmt.Println("Slice is sorted")
    }
    
    // Reverse
    slices.Reverse(nums)
    fmt.Println(nums)  // [9 6 5 4 3 2 1 1]
    
    // Remove duplicates from sorted slice
    nums = slices.Compact(nums)
    
    // Min and Max
    min := slices.Min(nums)
    max := slices.Max(nums)
    fmt.Printf("Min: %d, Max: %d\n", min, max)
    
    // Contains
    if slices.Contains(nums, 5) {
        fmt.Println("Contains 5")
    }
    
    // Index
    if i := slices.Index(nums, 3); i >= 0 {
        fmt.Printf("3 found at index %d\n", i)
    }
    
    // Equal
    other := []int{9, 6, 5, 4, 3, 2, 1}
    if slices.Equal(nums, other) {
        fmt.Println("Slices are equal")
    }
    
    // Clone
    clone := slices.Clone(nums)
    
    // Insert
    nums = slices.Insert(nums, 2, 99, 98)
    
    // Delete
    nums = slices.Delete(nums, 1, 3)
    
    // Replace
    nums = slices.Replace(nums, 1, 3, 77, 88)
    
    // Grow
    nums = slices.Grow(nums, 10)  // Ensure capacity for 10 more elements
    
    // Clip
    nums = slices.Clip(nums)  // Remove unused capacity
}

// Custom comparison functions
func customComparison() {
    type Person struct {
        Name string
        Age  int
    }
    
    people := []Person{
        {"Alice", 30},
        {"Bob", 25},
        {"Charlie", 35},
    }
    
    // Sort with custom comparison
    slices.SortFunc(people, func(a, b Person) int {
        return cmp.Compare(a.Age, b.Age)
    })
    
    // Sort stable (preserves order of equal elements)
    slices.SortStableFunc(people, func(a, b Person) int {
        return cmp.Compare(a.Name, b.Name)
    })
    
    // Binary search with custom comparison
    target := Person{Name: "Bob", Age: 25}
    i, found := slices.BinarySearchFunc(people, target, func(a, b Person) int {
        return cmp.Compare(a.Name, b.Name)
    })
    
    // Compare slices with custom function
    equal := slices.EqualFunc(people, people, func(a, b Person) bool {
        return a.Name == b.Name && a.Age == b.Age
    })
    
    // Compact with custom equality
    people = slices.CompactFunc(people, func(a, b Person) bool {
        return a.Name == b.Name
    })
}
```

## maps Package: Generic Map Operations

Map manipulation made easy:

```go
package main

import (
    "fmt"
    "maps"
)

func mapsOperations() {
    m1 := map[string]int{"a": 1, "b": 2, "c": 3}
    m2 := map[string]int{"b": 20, "d": 4}
    
    // Clone
    cloned := maps.Clone(m1)
    
    // Copy (merge m2 into m1)
    maps.Copy(m1, m2)
    fmt.Println(m1)  // map[a:1 b:20 c:3 d:4]
    
    // Equal
    if maps.Equal(m1, cloned) {
        fmt.Println("Maps are equal")
    }
    
    // DeleteFunc - delete entries matching predicate
    maps.DeleteFunc(m1, func(k string, v int) bool {
        return v < 10  // Delete all values less than 10
    })
    
    // Keys (Go 1.23+)
    keys := maps.Keys(m1)
    for k := range keys {
        fmt.Println(k)
    }
    
    // Values (Go 1.23+)
    values := maps.Values(m1)
    for v := range values {
        fmt.Println(v)
    }
    
    // EqualFunc - custom equality
    m3 := map[string]float64{"a": 1.0, "b": 2.0}
    m4 := map[string]float64{"a": 1.0001, "b": 2.0001}
    
    equal := maps.EqualFunc(m3, m4, func(v1, v2 float64) bool {
        return abs(v1-v2) < 0.001  // Fuzzy comparison
    })
}

// Collect keys and values (Go 1.23+)
func collectFromMaps() {
    m := map[string]int{"a": 1, "b": 2, "c": 3}
    
    // Collect all keys
    var keys []string
    for k := range maps.Keys(m) {
        keys = append(keys, k)
    }
    
    // Collect all values
    var values []int
    for v := range maps.Values(m) {
        values = append(values, v)
    }
    
    // Collect entries
    type Entry struct {
        Key   string
        Value int
    }
    
    var entries []Entry
    for k, v := range m {
        entries = append(entries, Entry{k, v})
    }
}
```

## cmp Package: Ordering and Comparison

Generic comparison utilities:

```go
package main

import (
    "cmp"
    "fmt"
    "slices"
)

func cmpOperations() {
    // Basic comparison
    result := cmp.Compare(5, 3)  // Returns 1 (5 > 3)
    
    // Or combinator - use first non-zero comparison
    finalResult := cmp.Or(
        cmp.Compare("a", "a"),  // 0 (equal)
        cmp.Compare(5, 3),      // 1 (first wins)
        cmp.Compare(1, 2),      // -1 (ignored)
    )
    fmt.Println(finalResult)  // 1
    
    // Less function for convenience
    if cmp.Less(3, 5) {
        fmt.Println("3 is less than 5")
    }
    
    // Ordered constraint for generics
    type Score[T cmp.Ordered] struct {
        Value T
        Name  string
    }
    
    scores := []Score[int]{
        {100, "Alice"},
        {85, "Bob"},
        {95, "Charlie"},
    }
    
    slices.SortFunc(scores, func(a, b Score[int]) int {
        return cmp.Compare(b.Value, a.Value)  // Descending
    })
}

// Complex sorting with cmp.Or
type Product struct {
    Category string
    Price    float64
    Name     string
    Stock    int
}

func sortProducts(products []Product) {
    slices.SortFunc(products, func(a, b Product) int {
        return cmp.Or(
            cmp.Compare(a.Category, b.Category),  // First by category
            cmp.Compare(a.Price, b.Price),        // Then by price
            cmp.Compare(a.Name, b.Name),          // Then by name
        )
    })
}

// Custom ordered types
func customOrdered[T cmp.Ordered](items []T) T {
    if len(items) == 0 {
        var zero T
        return zero
    }
    
    min := items[0]
    for _, item := range items[1:] {
        if cmp.Less(item, min) {
            min = item
        }
    }
    return min
}
```

## math/rand/v2: Better Randomness (Go 1.22+)

Improved random number generation:

```go
package main

import (
    "math/rand/v2"
    "fmt"
)

func modernRandom() {
    // No need to seed! Automatically seeded
    
    // Basic random numbers
    n := rand.IntN(100)        // [0, 100)
    f := rand.Float64()        // [0.0, 1.0)
    
    // New N-suffix methods (clearer API)
    i := rand.IntN(10)         // Replaces Intn
    ui := rand.UintN(10)       // New: unsigned
    i64 := rand.Int64N(1000)   // Type-specific
    
    // Shuffle any slice
    nums := []int{1, 2, 3, 4, 5}
    rand.Shuffle(len(nums), func(i, j int) {
        nums[i], nums[j] = nums[j], nums[i]
    })
    
    // Perm returns random permutation
    perm := rand.Perm(5)  // e.g., [2, 0, 4, 3, 1]
    
    // Custom random source
    src := rand.NewPCG(42, 1024)  // PCG algorithm with seed
    r := rand.New(src)
    value := r.IntN(100)
    
    // ChaCha8 for cryptographic randomness
    chacha := rand.NewChaCha8([32]byte{1, 2, 3})  // 32-byte seed
    secureRand := rand.New(chacha)
    secure := secureRand.Uint64()
}

// Weighted random selection
func weightedRandom() {
    items := []string{"common", "uncommon", "rare", "legendary"}
    weights := []float64{0.5, 0.3, 0.15, 0.05}
    
    // Calculate cumulative weights
    cumulative := make([]float64, len(weights))
    cumulative[0] = weights[0]
    for i := 1; i < len(weights); i++ {
        cumulative[i] = cumulative[i-1] + weights[i]
    }
    
    // Select based on weight
    r := rand.Float64()
    for i, c := range cumulative {
        if r < c {
            fmt.Printf("Selected: %s\n", items[i])
            break
        }
    }
}

// Concurrent random generation
func concurrentRandom() {
    // Each goroutine gets its own source
    for i := 0; i < 10; i++ {
        go func(id int) {
            // rand/v2 is safe for concurrent use
            value := rand.IntN(100)
            fmt.Printf("Goroutine %d: %d\n", id, value)
        }(i)
    }
}
```

## iter Package: Iterators (Go 1.23+)

Standard iterator patterns:

```go
package main

import (
    "iter"
    "fmt"
)

// Basic iterator types
type Iterator[T any] iter.Seq[T]
type Iterator2[K, V any] iter.Seq2[K, V]

// Custom iterator
func Range(start, end int) iter.Seq[int] {
    return func(yield func(int) bool) {
        for i := start; i < end; i++ {
            if !yield(i) {
                return
            }
        }
    }
}

// Using iterators with for-range
func useIterators() {
    // Range over custom iterator
    for v := range Range(1, 10) {
        fmt.Println(v)
    }
    
    // Early termination
    for v := range Range(1, 100) {
        if v > 10 {
            break  // Iterator stops
        }
        fmt.Println(v)
    }
}

// Two-value iterator
func Enumerate[T any](slice []T) iter.Seq2[int, T] {
    return func(yield func(int, T) bool) {
        for i, v := range slice {
            if !yield(i, v) {
                return
            }
        }
    }
}

// Filter iterator
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

// Map iterator
func Map[T, U any](seq iter.Seq[T], f func(T) U) iter.Seq[U] {
    return func(yield func(U) bool) {
        for v := range seq {
            if !yield(f(v)) {
                return
            }
        }
    }
}

// Chain iterators
func Chain[T any](seqs ...iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        for _, seq := range seqs {
            for v := range seq {
                if !yield(v) {
                    return
                }
            }
        }
    }
}

// Pull iterator (convert push to pull)
func pullExample() {
    next, stop := iter.Pull(Range(1, 10))
    defer stop()
    
    // Pull values manually
    if v, ok := next(); ok {
        fmt.Println("First:", v)
    }
    
    if v, ok := next(); ok {
        fmt.Println("Second:", v)
    }
}

// Tree iteration example
type Node[T any] struct {
    Value    T
    Children []*Node[T]
}

func (n *Node[T]) Iterate() iter.Seq[T] {
    return func(yield func(T) bool) {
        if !yield(n.Value) {
            return
        }
        
        for _, child := range n.Children {
            for v := range child.Iterate() {
                if !yield(v) {
                    return
                }
            }
        }
    }
}
```

## Additional Utility Packages

### max and min Built-ins (Go 1.21+)

```go
func builtinMinMax() {
    // Works with any ordered type
    a := min(3, 5, 1, 9, 2)  // 1
    b := max(3, 5, 1, 9, 2)  // 9
    
    // Works with strings
    s1 := min("apple", "banana", "cherry")  // "apple"
    s2 := max("apple", "banana", "cherry")  // "cherry"
    
    // Works with time
    t1 := time.Now()
    t2 := t1.Add(time.Hour)
    earliest := min(t1, t2)  // t1
}
```

### clear Built-in (Go 1.21+)

```go
func clearBuiltin() {
    // Clear map (keeps capacity)
    m := map[string]int{"a": 1, "b": 2}
    clear(m)  // m is now empty but retains capacity
    
    // Clear slice (zeros elements, keeps length/capacity)
    s := []int{1, 2, 3, 4, 5}
    clear(s)  // s is now [0, 0, 0, 0, 0]
}
```

## Experimental Packages Worth Watching

### golang.org/x/exp/slog (Before stdlib)

```go
// Many features tested in x/exp before stdlib adoption
import "golang.org/x/exp/slog"

// Features that graduated to log/slog
```

### golang.org/x/exp/slices (Before stdlib)

```go
// Experimental features that may come to stdlib
import "golang.org/x/exp/slices"

// Advanced operations being tested
func experimentalSlices() {
    // Chunk - split slice into chunks
    chunks := slices.Chunk([]int{1,2,3,4,5}, 2)
    // [[1,2], [3,4], [5]]
    
    // Permutations
    perms := slices.Permutations([]int{1,2,3})
    
    // GroupBy
    grouped := slices.GroupBy(items, func(item Item) string {
        return item.Category
    })
}
```

## Migration Guide

### From Old Random to rand/v2

```go
// Old (math/rand)
import "math/rand"
rand.Seed(time.Now().UnixNano())
n := rand.Intn(100)

// New (math/rand/v2)
import "math/rand/v2"
// No seed needed!
n := rand.IntN(100)
```

### From log to log/slog

```go
// Old
log.Printf("user logged in: id=%s time=%v", userID, time.Now())

// New
slog.Info("user logged in",
    "user_id", userID,
    "time", time.Now())
```

### From Manual Slice Operations to slices Package

```go
// Old: manual implementation
func contains(slice []string, item string) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// New: use slices package
found := slices.Contains(slice, item)
```

## Performance Comparisons

```go
func BenchmarkPackages(b *testing.B) {
    // Benchmark old vs new approaches
    
    b.Run("old-sort", func(b *testing.B) {
        data := make([]int, 1000)
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            sort.Ints(data)
        }
    })
    
    b.Run("new-slices-sort", func(b *testing.B) {
        data := make([]int, 1000)
        b.ResetTimer()
        for i := 0; i < b.N; i++ {
            slices.Sort(data)
        }
    })
    
    // slices.Sort is often faster due to generics
    // avoiding interface{} boxing
}
```

## Best Practices

### 1. Prefer Standard Packages
```go
// Use standard library when available
// Don't import third-party for what stdlib provides
```

### 2. Migrate Gradually
```go
// Start with new code
// Migrate old code as you touch it
```

### 3. Understand Performance Implications
```go
// Generic versions often faster (no interface{} boxing)
// But measure for your use case
```

### 4. Stay Updated
```go
// New packages are added regularly
// Check release notes
```

## Exercises

1. **Logger Wrapper**: Create a wrapper around slog that adds automatic tracing information.

2. **Custom Iterator**: Build an iterator for a custom data structure using the iter package.

3. **Slice Utilities**: Implement additional slice operations using generics that aren't in the slices package.

4. **Map Merger**: Create a sophisticated map merging function using the maps package.

5. **Random Testing**: Build a property-based testing framework using rand/v2.

## Summary

The new standard packages represent Go's evolution from a systems language to a complete platform. Each addition—from structured logging to generic collections—solves real problems identified through years of production use. These aren't just conveniences; they're foundations for the next generation of Go applications.

Key takeaways:
- log/slog brings production-ready structured logging
- slices and maps packages eliminate boilerplate
- cmp standardizes comparison operations  
- rand/v2 modernizes random number generation
- iter enables powerful iteration patterns

Next, we'll explore how existing packages have been enhanced with new features and capabilities.

---

*Continue to [Chapter 13: Enhanced Existing Packages](chapter13.md)*