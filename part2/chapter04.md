# Chapter 4: Generic Programming in Practice

## From Theory to Reality

Now that you understand type parameters, let's explore how they're actually used in production Go code. After three years of generics in the wild, patterns have emerged, mistakes have been made, and wisdom has been earned. This chapter distills that collective experience.

## The Standard Library's Generic Evolution

The Go team led by example, carefully adding generic APIs to the standard library. These additions show us the "Go way" of using generics.

### The slices Package

The `slices` package (introduced in Go 1.21) provides generic functions for slice operations:

```go
package main

import (
    "cmp"
    "fmt"
    "slices"
)

func main() {
    numbers := []int{3, 1, 4, 1, 5, 9, 2, 6}
    
    // Sorting
    slices.Sort(numbers)
    fmt.Println(numbers)  // [1 1 2 3 4 5 6 9]
    
    // Binary search (on sorted slice)
    idx, found := slices.BinarySearch(numbers, 5)
    fmt.Printf("5 found at index %d: %v\n", idx, found)
    
    // Reversing
    slices.Reverse(numbers)
    fmt.Println(numbers)  // [9 6 5 4 3 2 1 1]
    
    // Removing duplicates
    numbers = slices.Compact(numbers)
    
    // Checking equality
    other := []int{9, 6, 5, 4, 3, 2, 1}
    equal := slices.Equal(numbers, other)
    fmt.Printf("Slices equal: %v\n", equal)
    
    // Custom comparison
    people := []Person{
        {Name: "Alice", Age: 30},
        {Name: "Bob", Age: 25},
        {Name: "Charlie", Age: 35},
    }
    
    slices.SortFunc(people, func(a, b Person) int {
        return cmp.Compare(a.Age, b.Age)
    })
}

type Person struct {
    Name string
    Age  int
}
```

### The maps Package

The `maps` package provides generic map operations:

```go
package main

import (
    "fmt"
    "maps"
)

func main() {
    m1 := map[string]int{"a": 1, "b": 2}
    m2 := map[string]int{"b": 3, "c": 4}
    
    // Cloning
    backup := maps.Clone(m1)
    
    // Merging (m2 into m1)
    maps.Copy(m1, m2)
    fmt.Println(m1)  // map[a:1 b:3 c:4]
    
    // Equality check
    equal := maps.Equal(m1, backup)
    fmt.Printf("Maps equal: %v\n", equal)  // false
    
    // Delete with predicate
    maps.DeleteFunc(m1, func(k string, v int) bool {
        return v < 2
    })
    
    // Collecting keys and values
    keys := maps.Keys(m1)
    values := maps.Values(m1)
    
    fmt.Printf("Keys: %v, Values: %v\n", keys, values)
}
```

### The cmp Package

The `cmp` package provides ordering utilities:

```go
package main

import (
    "cmp"
    "fmt"
)

func main() {
    // Basic comparison
    result := cmp.Compare(5, 3)  // returns 1 (5 > 3)
    
    // Using Or for fallback comparisons
    comparePersons := func(a, b Person) int {
        return cmp.Or(
            cmp.Compare(a.LastName, b.LastName),   // First by last name
            cmp.Compare(a.FirstName, b.FirstName), // Then by first name
            cmp.Compare(a.Age, b.Age),              // Finally by age
        )
    }
    
    p1 := Person{FirstName: "John", LastName: "Doe", Age: 30}
    p2 := Person{FirstName: "Jane", LastName: "Doe", Age: 25}
    
    fmt.Println(comparePersons(p1, p2))  // 1 (John > Jane)
}
```

## Real-World Generic Patterns

### Pattern 1: Option Types

Go doesn't have built-in option types, but generics make them practical:

```go
type Option[T any] struct {
    value *T
}

func Some[T any](value T) Option[T] {
    return Option[T]{value: &value}
}

func None[T any]() Option[T] {
    return Option[T]{value: nil}
}

func (o Option[T]) IsSome() bool {
    return o.value != nil
}

func (o Option[T]) IsNone() bool {
    return o.value == nil
}

func (o Option[T]) Unwrap() T {
    if o.value == nil {
        panic("unwrap on None")
    }
    return *o.value
}

func (o Option[T]) UnwrapOr(defaultValue T) T {
    if o.value == nil {
        return defaultValue
    }
    return *o.value
}

func (o Option[T]) Map[U any](fn func(T) U) Option[U] {
    if o.value == nil {
        return None[U]()
    }
    return Some(fn(*o.value))
}

// Usage
func findUser(id int) Option[User] {
    if user, exists := database[id]; exists {
        return Some(user)
    }
    return None[User]()
}

func main() {
    userOpt := findUser(123)
    
    name := userOpt.
        Map(func(u User) string { return u.Name }).
        UnwrapOr("Unknown User")
    
    fmt.Println(name)
}
```

### Pattern 2: Result Types for Error Handling

A Result type can make error handling more functional:

```go
type Result[T any] struct {
    value *T
    err   error
}

func Ok[T any](value T) Result[T] {
    return Result[T]{value: &value}
}

func Err[T any](err error) Result[T] {
    return Result[T]{err: err}
}

func (r Result[T]) IsOk() bool {
    return r.err == nil
}

func (r Result[T]) IsErr() bool {
    return r.err != nil
}

func (r Result[T]) Unwrap() T {
    if r.err != nil {
        panic(fmt.Sprintf("unwrap on Err: %v", r.err))
    }
    return *r.value
}

func (r Result[T]) UnwrapErr() error {
    return r.err
}

func (r Result[T]) Map[U any](fn func(T) U) Result[U] {
    if r.err != nil {
        return Err[U](r.err)
    }
    return Ok(fn(*r.value))
}

func (r Result[T]) AndThen[U any](fn func(T) Result[U]) Result[U] {
    if r.err != nil {
        return Err[U](r.err)
    }
    return fn(*r.value)
}

// Usage
func parseNumber(s string) Result[int] {
    n, err := strconv.Atoi(s)
    if err != nil {
        return Err[int](err)
    }
    return Ok(n)
}

func doubleIfPositive(n int) Result[int] {
    if n < 0 {
        return Err[int](errors.New("negative number"))
    }
    return Ok(n * 2)
}

func main() {
    result := parseNumber("42").
        AndThen(doubleIfPositive).
        Map(func(n int) string { return fmt.Sprintf("Result: %d", n) })
    
    if result.IsOk() {
        fmt.Println(result.Unwrap())
    } else {
        fmt.Printf("Error: %v\n", result.UnwrapErr())
    }
}
```

### Pattern 3: Builder Pattern with Generics

Generics enable type-safe builders:

```go
type Builder[T any] struct {
    value   T
    errors  []error
}

func NewBuilder[T any](initial T) *Builder[T] {
    return &Builder[T]{value: initial}
}

func (b *Builder[T]) Apply(fn func(*T) error) *Builder[T] {
    if len(b.errors) > 0 {
        return b  // Skip if already failed
    }
    
    if err := fn(&b.value); err != nil {
        b.errors = append(b.errors, err)
    }
    return b
}

func (b *Builder[T]) Build() (T, error) {
    if len(b.errors) > 0 {
        return b.value, errors.Join(b.errors...)
    }
    return b.value, nil
}

// Usage
type Config struct {
    Host     string
    Port     int
    Timeout  time.Duration
    MaxConns int
}

func main() {
    config, err := NewBuilder(Config{}).
        Apply(func(c *Config) error {
            c.Host = "localhost"
            return nil
        }).
        Apply(func(c *Config) error {
            port, err := strconv.Atoi(os.Getenv("PORT"))
            if err != nil {
                return fmt.Errorf("invalid port: %w", err)
            }
            c.Port = port
            return nil
        }).
        Apply(func(c *Config) error {
            c.Timeout = 30 * time.Second
            c.MaxConns = 100
            return nil
        }).
        Build()
    
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("Config: %+v\n", config)
}
```

### Pattern 4: Generic Middleware

Create reusable middleware patterns:

```go
type Middleware[T any] func(Handler[T]) Handler[T]
type Handler[T any] func(context.Context, T) error

func Chain[T any](middlewares ...Middleware[T]) Middleware[T] {
    return func(next Handler[T]) Handler[T] {
        for i := len(middlewares) - 1; i >= 0; i-- {
            next = middlewares[i](next)
        }
        return next
    }
}

func WithLogging[T any]() Middleware[T] {
    return func(next Handler[T]) Handler[T] {
        return func(ctx context.Context, req T) error {
            slog.Info("handling request", "type", fmt.Sprintf("%T", req))
            err := next(ctx, req)
            if err != nil {
                slog.Error("request failed", "error", err)
            }
            return err
        }
    }
}

func WithTimeout[T any](timeout time.Duration) Middleware[T] {
    return func(next Handler[T]) Handler[T] {
        return func(ctx context.Context, req T) error {
            ctx, cancel := context.WithTimeout(ctx, timeout)
            defer cancel()
            return next(ctx, req)
        }
    }
}

func WithRetry[T any](attempts int) Middleware[T] {
    return func(next Handler[T]) Handler[T] {
        return func(ctx context.Context, req T) error {
            var err error
            for i := 0; i < attempts; i++ {
                if err = next(ctx, req); err == nil {
                    return nil
                }
                if i < attempts-1 {
                    time.Sleep(time.Second * time.Duration(i+1))
                }
            }
            return fmt.Errorf("after %d attempts: %w", attempts, err)
        }
    }
}

// Usage
type EmailRequest struct {
    To      string
    Subject string
    Body    string
}

func sendEmail(ctx context.Context, req EmailRequest) error {
    // Actual email sending logic
    fmt.Printf("Sending email to %s\n", req.To)
    return nil
}

func main() {
    handler := Chain[EmailRequest](
        WithLogging[EmailRequest](),
        WithTimeout[EmailRequest](5*time.Second),
        WithRetry[EmailRequest](3),
    )(sendEmail)
    
    ctx := context.Background()
    req := EmailRequest{
        To:      "user@example.com",
        Subject: "Hello",
        Body:    "World",
    }
    
    if err := handler(ctx, req); err != nil {
        log.Fatal(err)
    }
}
```

### Pattern 5: Generic Sync Primitives

Build thread-safe generic containers:

```go
type SyncMap[K comparable, V any] struct {
    mu sync.RWMutex
    m  map[K]V
}

func NewSyncMap[K comparable, V any]() *SyncMap[K, V] {
    return &SyncMap[K, V]{
        m: make(map[K]V),
    }
}

func (sm *SyncMap[K, V]) Load(key K) (V, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

func (sm *SyncMap[K, V]) Store(key K, value V) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}

func (sm *SyncMap[K, V]) LoadOrStore(key K, value V) (actual V, loaded bool) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    
    if val, ok := sm.m[key]; ok {
        return val, true
    }
    
    sm.m[key] = value
    return value, false
}

func (sm *SyncMap[K, V]) Delete(key K) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    delete(sm.m, key)
}

func (sm *SyncMap[K, V]) Range(fn func(key K, value V) bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    
    for k, v := range sm.m {
        if !fn(k, v) {
            break
        }
    }
}

// Generic channel-based queue
type Queue[T any] struct {
    items chan T
}

func NewQueue[T any](capacity int) *Queue[T] {
    return &Queue[T]{
        items: make(chan T, capacity),
    }
}

func (q *Queue[T]) Enqueue(item T) error {
    select {
    case q.items <- item:
        return nil
    default:
        return errors.New("queue full")
    }
}

func (q *Queue[T]) Dequeue() (T, error) {
    select {
    case item := <-q.items:
        return item, nil
    default:
        var zero T
        return zero, errors.New("queue empty")
    }
}

func (q *Queue[T]) DequeueWithTimeout(timeout time.Duration) (T, error) {
    select {
    case item := <-q.items:
        return item, nil
    case <-time.After(timeout):
        var zero T
        return zero, errors.New("timeout")
    }
}
```

## Generic Algorithms

### Functional Programming Patterns

Generics enable functional programming patterns in Go:

```go
// Map transforms each element
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Filter selects elements matching a predicate
func Filter[T any](slice []T, predicate func(T) bool) []T {
    result := make([]T, 0, len(slice))
    for _, v := range slice {
        if predicate(v) {
            result = append(result, v)
        }
    }
    return result
}

// Reduce aggregates elements
func Reduce[T, U any](slice []T, initial U, fn func(U, T) U) U {
    result := initial
    for _, v := range slice {
        result = fn(result, v)
    }
    return result
}

// FlatMap maps and flattens
func FlatMap[T, U any](slice []T, fn func(T) []U) []U {
    var result []U
    for _, v := range slice {
        result = append(result, fn(v)...)
    }
    return result
}

// Partition splits based on predicate
func Partition[T any](slice []T, predicate func(T) bool) ([]T, []T) {
    var pass, fail []T
    for _, v := range slice {
        if predicate(v) {
            pass = append(pass, v)
        } else {
            fail = append(fail, v)
        }
    }
    return pass, fail
}

// Usage example
func main() {
    numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
    
    // Chain operations
    result := Map(
        Filter(numbers, func(n int) bool { return n%2 == 0 }),
        func(n int) string { return fmt.Sprintf("even-%d", n) },
    )
    fmt.Println(result)  // [even-2 even-4 even-6 even-8 even-10]
    
    // Reduce to sum
    sum := Reduce(numbers, 0, func(acc, n int) int { return acc + n })
    fmt.Println(sum)  // 55
    
    // Partition evens and odds
    evens, odds := Partition(numbers, func(n int) bool { return n%2 == 0 })
    fmt.Printf("Evens: %v, Odds: %v\n", evens, odds)
}
```

### Tree and Graph Algorithms

Generic tree traversal:

```go
type TreeNode[T any] struct {
    Value    T
    Children []*TreeNode[T]
}

func (n *TreeNode[T]) DFS(visit func(T)) {
    if n == nil {
        return
    }
    visit(n.Value)
    for _, child := range n.Children {
        child.DFS(visit)
    }
}

func (n *TreeNode[T]) BFS(visit func(T)) {
    if n == nil {
        return
    }
    
    queue := []*TreeNode[T]{n}
    for len(queue) > 0 {
        current := queue[0]
        queue = queue[1:]
        
        visit(current.Value)
        queue = append(queue, current.Children...)
    }
}

func (n *TreeNode[T]) Find(predicate func(T) bool) *TreeNode[T] {
    if n == nil {
        return nil
    }
    
    if predicate(n.Value) {
        return n
    }
    
    for _, child := range n.Children {
        if found := child.Find(predicate); found != nil {
            return found
        }
    }
    
    return nil
}

func (n *TreeNode[T]) Map[U any](fn func(T) U) *TreeNode[U] {
    if n == nil {
        return nil
    }
    
    result := &TreeNode[U]{
        Value:    fn(n.Value),
        Children: make([]*TreeNode[U], len(n.Children)),
    }
    
    for i, child := range n.Children {
        result.Children[i] = child.Map(fn)
    }
    
    return result
}
```

## Advanced Generic Techniques

### Type Constraints Composition

Building complex constraints from simple ones:

```go
// Basic constraints
type Numeric interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}

type Signed interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 | ~float32 | ~float64
}

type Unsigned interface {
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64
}

// Composite constraint
type SignedNumeric interface {
    Numeric
    Signed
}

// Complex constraint with methods
type Measurable[T any] interface {
    Measure() T
    comparable
}

type Container[T Measurable[int]] struct {
    items []T
}

func (c *Container[T]) TotalSize() int {
    total := 0
    for _, item := range c.items {
        total += item.Measure()
    }
    return total
}
```

### Self-Referential Generics

Types that reference themselves generically:

```go
type Comparable[T any] interface {
    CompareTo(T) int
}

type SortableList[T Comparable[T]] []T

func (s SortableList[T]) Len() int           { return len(s) }
func (s SortableList[T]) Less(i, j int) bool { return s[i].CompareTo(s[j]) < 0 }
func (s SortableList[T]) Swap(i, j int)      { s[i], s[j] = s[j], s[i] }

// Usage
type Version struct {
    Major, Minor, Patch int
}

func (v Version) CompareTo(other Version) int {
    if v.Major != other.Major {
        return v.Major - other.Major
    }
    if v.Minor != other.Minor {
        return v.Minor - other.Minor
    }
    return v.Patch - other.Patch
}

func main() {
    versions := SortableList[Version]{
        {1, 2, 3},
        {1, 0, 0},
        {2, 0, 0},
        {1, 1, 0},
    }
    
    sort.Sort(versions)
    fmt.Println(versions)  // [{1 0 0} {1 1 0} {1 2 3} {2 0 0}]
}
```

### Phantom Types

Using type parameters for compile-time guarantees without runtime representation:

```go
type State interface {
    state()
}

type Open struct{}
func (Open) state() {}

type Closed struct{}
func (Closed) state() {}

type File[S State] struct {
    path string
    // S is never used in fields, only for type safety
}

func OpenFile(path string) *File[Open] {
    return &File[Open]{path: path}
}

func (f *File[Open]) Read() ([]byte, error) {
    // Can only read open files
    return os.ReadFile(f.path)
}

func (f *File[Open]) Close() *File[Closed] {
    // Transition from Open to Closed
    return &File[Closed]{path: f.path}
}

func (f *File[Closed]) Open() *File[Open] {
    // Transition from Closed to Open
    return &File[Open]{path: f.path}
}

// This won't compile:
// func (f *File[Closed]) Read() ([]byte, error) {
//     // Can't read closed files!
// }

func main() {
    file := OpenFile("data.txt")
    data, _ := file.Read()  // OK: file is open
    
    closed := file.Close()
    // data, _ = closed.Read()  // Compile error: no Read method
    
    reopened := closed.Open()
    data, _ = reopened.Read()  // OK again
}
```

## Performance Patterns

### Zero-Allocation Generics

Design generic code that avoids allocations:

```go
// Pool for reusable objects
type Pool[T any] struct {
    pool sync.Pool
    new  func() T
}

func NewPool[T any](new func() T) *Pool[T] {
    return &Pool[T]{
        pool: sync.Pool{
            New: func() interface{} { return new() },
        },
        new: new,
    }
}

func (p *Pool[T]) Get() T {
    return p.pool.Get().(T)
}

func (p *Pool[T]) Put(v T) {
    p.pool.Put(v)
}

// Buffer pool example
var bufferPool = NewPool(func() *bytes.Buffer {
    return new(bytes.Buffer)
})

func ProcessData(data []byte) string {
    buf := bufferPool.Get()
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    buf.Write(data)
    // Process...
    return buf.String()
}
```

### Specialized Implementations

Use type parameters to generate specialized code:

```go
type NumericSlice[T Numeric] []T

func (s NumericSlice[T]) Sum() T {
    var sum T
    for _, v := range s {
        sum += v
    }
    return sum
}

func (s NumericSlice[T]) Product() T {
    if len(s) == 0 {
        return 0
    }
    product := s[0]
    for _, v := range s[1:] {
        product *= v
    }
    return product
}

// Specialized for better performance than interface{}
func (s NumericSlice[T]) Average() float64 {
    if len(s) == 0 {
        return 0
    }
    return float64(s.Sum()) / float64(len(s))
}
```

## Testing Generic Code

### Table-Driven Tests with Generics

```go
func TestContainer(t *testing.T) {
    type testCase[T any] struct {
        name     string
        initial  []T
        add      T
        expected []T
    }
    
    runTests := func(t *testing.T, cases []testCase[int]) {
        for _, tc := range cases {
            t.Run(tc.name, func(t *testing.T) {
                c := NewContainer[int]()
                for _, v := range tc.initial {
                    c.Add(v)
                }
                c.Add(tc.add)
                
                got := c.Items()
                if !slices.Equal(got, tc.expected) {
                    t.Errorf("got %v, want %v", got, tc.expected)
                }
            })
        }
    }
    
    intCases := []testCase[int]{
        {
            name:     "add to empty",
            initial:  []int{},
            add:      42,
            expected: []int{42},
        },
        {
            name:     "add to non-empty",
            initial:  []int{1, 2},
            add:      3,
            expected: []int{1, 2, 3},
        },
    }
    
    runTests(t, intCases)
}
```

### Property-Based Testing

Use generics for property-based testing:

```go
func PropertyReverseTwiceIsOriginal[T any](t *testing.T, gen func() []T) {
    for i := 0; i < 100; i++ {
        original := gen()
        reversed := slices.Clone(original)
        slices.Reverse(reversed)
        slices.Reverse(reversed)
        
        if !slices.Equal(original, reversed) {
            t.Errorf("reverse twice failed: %v != %v", original, reversed)
        }
    }
}

func TestProperties(t *testing.T) {
    PropertyReverseTwiceIsOriginal(t, func() []int {
        n := rand.Intn(100)
        slice := make([]int, n)
        for i := range slice {
            slice[i] = rand.Intn(1000)
        }
        return slice
    })
}
```

## Migration Strategies

### Identifying Generic Opportunities

Look for these patterns in existing code:

```go
// 1. Duplicate functions for different types
func ContainsInt(slice []int, item int) bool { /*...*/ }
func ContainsString(slice []string, item string) bool { /*...*/ }

// 2. Interface{} with type assertions
func Contains(slice []interface{}, item interface{}) bool {
    for _, v := range slice {
        if v == item {  // Requires same type
            return true
        }
    }
    return false
}

// 3. Code generation comments
//go:generate gen type=int,string,float64

// Replace all with:
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}
```

### Gradual Migration

Start with internal APIs before public ones:

```go
// Phase 1: Internal generic implementation
func containsGeneric[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// Phase 2: Existing APIs delegate to generic
func ContainsInt(slice []int, item int) bool {
    return containsGeneric(slice, item)
}

func ContainsString(slice []string, item string) bool {
    return containsGeneric(slice, item)
}

// Phase 3: Deprecate old APIs
// Deprecated: Use Contains instead
func ContainsInt(slice []int, item int) bool {
    return Contains(slice, item)
}

// Phase 4: Remove deprecated APIs in major version
```

## Best Practices Refined

### 1. Don't Over-Generalize
```go
// Too generic - harder to understand and use
func Process[T, U, V any](t T, fn func(T) U, fn2 func(U) V) V

// Better - specific and clear
func MapThenFilter[T any](slice []T, mapFn func(T) T, filterFn func(T) bool) []T
```

### 2. Provide Non-Generic Wrappers When Helpful
```go
// Generic implementation
func Sort[T cmp.Ordered](slice []T) {
    slices.Sort(slice)
}

// Convenience wrappers for common cases
func SortInts(slice []int) {
    Sort(slice)
}

func SortStrings(slice []string) {
    Sort(slice)
}
```

### 3. Use Meaningful Constraint Names
```go
// Poor
func Process[T interface{ ~int | ~string }](v T)

// Better
type IntOrString interface {
    ~int | ~string
}

func Process[T IntOrString](v T)
```

### 4. Document Type Parameter Requirements
```go
// Sum returns the sum of all elements in the slice.
// T must be a numeric type that supports the + operator.
func Sum[T Numeric](slice []T) T {
    var sum T
    for _, v := range slice {
        sum += v
    }
    return sum
}
```

## Exercises

1. **Generic LRU Cache**: Implement a generic LRU cache with a configurable size.

2. **Pipeline Builder**: Create a generic pipeline that can chain operations:
   ```go
   result := NewPipeline(data).
       Stage(transform1).
       Stage(transform2).
       Filter(predicate).
       Execute()
   ```

3. **Generic Event Bus**: Build a type-safe event bus that handles different event types.

4. **Retry with Backoff**: Implement a generic retry mechanism with exponential backoff.

5. **Type-Safe SQL Builder**: Create a generic SQL query builder that ensures type safety.

## Summary

Generic programming in Go has matured from an experiment to an essential tool. The patterns in this chapter represent the collective wisdom of the Go community—patterns that solve real problems while maintaining Go's simplicity.

Key takeaways:
- Start with concrete types, extract generic patterns when clear
- The standard library shows the "Go way" of using generics
- Generics shine for containers, algorithms, and type-safe APIs
- Don't force generics where they don't add value
- Performance is generally excellent with proper design

Next, we'll explore how error handling has evolved beyond simple error returns, with wrapping, inspection, and modern patterns that make Go errors more powerful and informative.

---

*Continue to [Chapter 5: Error Handling Evolution](../part3/chapter05.md)*