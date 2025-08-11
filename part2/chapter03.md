# Chapter 3: Understanding Type Parameters

## The Long Road to Generics

For thirteen years, Go stubbornly resisted generics. This wasn't oversight—it was philosophy. The Go team believed that most problems didn't need generics, and adding them would complicate a language built on simplicity. They were partially right. But they were also partially wrong.

The community's workarounds told the story:

```go
// Before generics: The interface{} shuffle
func Contains(slice []interface{}, item interface{}) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// Using it was... awkward
numbers := []int{1, 2, 3}
// Can't pass []int to []interface{} directly!
interfaceSlice := make([]interface{}, len(numbers))
for i, v := range numbers {
    interfaceSlice[i] = v
}
found := Contains(interfaceSlice, 2)  // Finally!
```

Or the code generation approach:

```bash
# Generate type-specific versions
$ go generate ./...
# Created: contains_int.go, contains_string.go, contains_float64.go...
```

In March 2022, Go 1.18 finally introduced generics. Not because the team changed their mind about simplicity, but because they found a way to add generics that felt like Go.

## Your First Generic Function

Let's start with the function everyone writes first:

```go
// Generic function with type parameter T
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// Using it is natural
numbers := []int{1, 2, 3}
found := Contains(numbers, 2)  // true

words := []string{"go", "rust", "python"}
hasGo := Contains(words, "go")  // true

// Type inference just works
// No need to write Contains[int](numbers, 2)
```

Let's break down the syntax:

- `[T comparable]` - Type parameter list
- `T` - Type parameter name (like a variable for types)
- `comparable` - Type constraint (what T must satisfy)
- `[]T` - Using T like any other type

## Type Parameters: The Basics

Type parameters let you write code that works with multiple types while maintaining type safety. They can appear on functions and types:

```go
// Function with single type parameter
func First[T any](slice []T) (T, bool) {
    if len(slice) == 0 {
        var zero T
        return zero, false
    }
    return slice[0], true
}

// Function with multiple type parameters
func Map[T, U any](slice []T, fn func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = fn(v)
    }
    return result
}

// Type with type parameters
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 {
        var zero T
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

## Type Constraints: Setting Boundaries

Type constraints define what types can be used for type parameters. Go provides several built-in constraints:

### The `any` Constraint

The most permissive constraint, allowing any type:

```go
func Print[T any](value T) {
    fmt.Println(value)
}

// Works with anything
Print(42)
Print("hello")
Print([]int{1, 2, 3})
Print(struct{ Name string }{"Alice"})
```

### The `comparable` Constraint

Types that support `==` and `!=`:

```go
func Equal[T comparable](a, b T) bool {
    return a == b
}

// Works with comparable types
Equal(1, 1)           // true
Equal("go", "go")     // true
Equal(3.14, 3.14)     // true

// Doesn't work with non-comparable types
// Equal([]int{1}, []int{1})  // Error: []int is not comparable
// Equal(func(){}, func(){})   // Error: func() is not comparable
```

### Custom Constraints

You can define your own constraints using interfaces:

```go
// Constraint for types that can be ordered
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 |
    ~string
}

func Max[T Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

// The standard library provides this as cmp.Ordered
import "cmp"

func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

## Type Sets and Union Types

Constraints work through type sets—the set of types that satisfy the constraint:

```go
// Union type constraint
type Number interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}

func Sum[T Number](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}

// The ~ prefix means "underlying type"
type MyInt int

var nums []MyInt = []MyInt{1, 2, 3}
total := Sum(nums)  // Works! MyInt's underlying type is int
```

### Constraint Literals

You can use constraints inline without naming them:

```go
func Double[T ~int | ~float64](x T) T {
    return x * 2
}

// Equivalent to:
type Numeric interface {
    ~int | ~float64
}

func Double2[T Numeric](x T) T {
    return x * 2
}
```

## Type Inference: Smart Defaults

Go's type inference for generics is sophisticated:

```go
// Full explicit form
result := Map[int, string]([]int{1, 2, 3}, strconv.Itoa)

// Type inference from arguments
result := Map([]int{1, 2, 3}, strconv.Itoa)  // T=int, U=string inferred

// Partial inference
type Pair[T, U any] struct {
    First  T
    Second U
}

// Must specify both types (no arguments to infer from)
p := Pair[int, string]{First: 1, Second: "one"}

// Helper function for inference
func NewPair[T, U any](first T, second U) Pair[T, U] {
    return Pair[T, U]{First: first, Second: second}
}

p := NewPair(1, "one")  // Types inferred!
```

### Inference Limitations

Type inference has limits:

```go
// Cannot infer from return type alone
func Zero[T any]() T {
    var zero T
    return zero
}

// Must specify type
var x int = Zero[int]()  // Can't infer T from assignment

// Cannot infer from nil
func ProcessPtr[T any](p *T) {
    // ...
}

// ProcessPtr(nil)  // Error: cannot infer T
ProcessPtr[string](nil)  // Must specify
```

## Methods and Type Parameters

Generic types can have methods, but with restrictions:

```go
type Container[T any] struct {
    items []T
}

// Methods can use the type parameter
func (c *Container[T]) Add(item T) {
    c.items = append(c.items, item)
}

// Methods CANNOT have their own type parameters
// func (c *Container[T]) Convert[U any](fn func(T) U) Container[U]  // ERROR!

// Use a function instead
func Convert[T, U any](c Container[T], fn func(T) U) Container[U] {
    result := Container[U]{}
    for _, item := range c.items {
        result.Add(fn(item))
    }
    return result
}
```

## Constraints with Methods

Constraints can require methods, not just types:

```go
type Stringer interface {
    String() string
}

func Join[T Stringer](items []T, sep string) string {
    var parts []string
    for _, item := range items {
        parts = append(parts, item.String())
    }
    return strings.Join(parts, sep)
}

// More complex constraint
type Numeric interface {
    ~int | ~int32 | ~int64 | ~float32 | ~float64
}

type Calculator[T Numeric] interface {
    Add(T, T) T
    Multiply(T, T) T
}
```

## Type Parameters in Practice

### The Zero Value Problem

Getting zero values in generic code:

```go
func GetOrDefault[T any](m map[string]T, key string) T {
    if val, ok := m[key]; ok {
        return val
    }
    var zero T  // Zero value of T
    return zero
}

// Or use named return
func GetOrDefault2[T any](m map[string]T, key string) (result T) {
    if val, ok := m[key]; ok {
        return val
    }
    return  // Returns zero value
}
```

### The Pointer Pattern

Working with pointers generically:

```go
func ToPtr[T any](v T) *T {
    return &v
}

func FromPtr[T any](p *T) T {
    if p == nil {
        var zero T
        return zero
    }
    return *p
}

// Usage
strPtr := ToPtr("hello")
value := FromPtr(strPtr)  // "hello"
value2 := FromPtr[string](nil)  // ""
```

### Generic Containers

Building reusable data structures:

```go
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{
        items: make(map[T]struct{}),
    }
}

func (s *Set[T]) Add(item T) {
    s.items[item] = struct{}{}
}

func (s *Set[T]) Contains(item T) bool {
    _, exists := s.items[item]
    return exists
}

func (s *Set[T]) Remove(item T) {
    delete(s.items, item)
}

func (s *Set[T]) Size() int {
    return len(s.items)
}

// Usage
numbers := NewSet[int]()
numbers.Add(1)
numbers.Add(2)
numbers.Add(1)  // Duplicate, ignored
fmt.Println(numbers.Size())  // 2
```

## Advanced Constraint Techniques

### Embedded Constraints

Constraints can embed other constraints:

```go
type Ordered interface {
    ~int | ~float64 | ~string
}

type ComparableOrdered interface {
    comparable
    Ordered
}

func FindMin[T ComparableOrdered](slice []T) (T, bool) {
    if len(slice) == 0 {
        var zero T
        return zero, false
    }
    
    min := slice[0]
    for _, v := range slice[1:] {
        if v < min {
            min = v
        }
    }
    return min, true
}
```

### Self-Referential Constraints

Constraints can reference the type parameter:

```go
type Addable[T any] interface {
    Add(T) T
}

func Sum[T Addable[T]](values []T) T {
    if len(values) == 0 {
        var zero T
        return zero
    }
    
    sum := values[0]
    for _, v := range values[1:] {
        sum = sum.Add(v)
    }
    return sum
}
```

### Mutually Referential Type Parameters

Type parameters can reference each other:

```go
type Graph[Node any, Edge any] struct {
    nodes []Node
    edges []Edge
}

type WeightedGraph[N comparable, E WeightedEdge[N]] struct {
    Graph[N, E]
}

type WeightedEdge[N comparable] struct {
    From, To N
    Weight   float64
}
```

## Type Parameter Lists

Functions and types can have multiple type parameters:

```go
func Zip[T, U any](ts []T, us []U) []struct{T T; U U} {
    minLen := len(ts)
    if len(us) < minLen {
        minLen = len(us)
    }
    
    result := make([]struct{T T; U U}, minLen)
    for i := 0; i < minLen; i++ {
        result[i] = struct{T T; U U}{T: ts[i], U: us[i]}
    }
    return result
}

// Multiple constraints
func Process[T comparable, U any, V ~int](t T, u U, v V) {
    // Can use T with ==
    // U can be anything
    // V must have underlying type int
}
```

## Performance Considerations

### Compilation Strategy

Go implements generics through "GCShape stenciling":

```go
// These share the same compiled code (same GCShape)
func Print[T any](v T) { fmt.Println(v) }

Print(42)       // int
Print(int32(42)) // int32
Print(true)     // bool
// All use same shape for scalar types

Print("hello")           // string
Print([]int{1, 2, 3})   // []int
// Different shapes for different memory layouts
```

This means:
- Moderate binary size increase
- Good runtime performance
- Some compile-time overhead

### When Generics Help Performance

```go
// Generic version avoids interface boxing
func SumGeneric[T ~int | ~float64](nums []T) T {
    var sum T
    for _, n := range nums {
        sum += n
    }
    return sum
}

// Interface version requires boxing/unboxing
func SumInterface(nums []interface{}) float64 {
    var sum float64
    for _, n := range nums {
        sum += n.(float64)  // Runtime type assertion
    }
    return sum
}

// Generic is faster: no boxing, no type assertions
```

## Common Pitfalls

### Pitfall 1: Over-Constraining

```go
// Too restrictive
func Process[T comparable](items []T) []T {
    // Only uses len and indexing, doesn't need comparable
    result := make([]T, len(items))
    copy(result, items)
    return result
}

// Better
func Process[T any](items []T) []T {
    result := make([]T, len(items))
    copy(result, items)
    return result
}
```

### Pitfall 2: Type Parameter in Wrong Scope

```go
type Container[T any] struct {
    items []T
}

// Wrong: Method can't have type parameters
// func (c Container[T]) Map[U any](fn func(T) U) Container[U]

// Right: Use package-level function
func MapContainer[T, U any](c Container[T], fn func(T) U) Container[U] {
    // ...
}
```

### Pitfall 3: Forgetting Type Arguments

```go
// Declaring variables of generic types
var s Set[int]  // Must specify type argument

// In structs
type MyStruct struct {
    // numbers Set  // Error: missing type argument
    numbers Set[int]  // Correct
}
```

## Best Practices

### 1. Start Concrete, Generalize Later
Write the concrete version first, then extract the generic pattern:

```go
// Start with
func ContainsInt(slice []int, item int) bool { /*...*/ }
func ContainsString(slice []string, item string) bool { /*...*/ }

// Extract pattern
func Contains[T comparable](slice []T, item T) bool { /*...*/ }
```

### 2. Use Meaningful Type Parameter Names
- `T` for single, general types
- `K, V` for key-value pairs
- `E` for elements
- Descriptive names for domain-specific types

```go
func Merge[Key comparable, Value any](m1, m2 map[Key]Value) map[Key]Value
```

### 3. Prefer Functions Over Methods
Since methods can't have type parameters, prefer functions for generic operations:

```go
// Instead of wanting: container.Map[U](fn)
// Use: Map(container, fn)
```

### 4. Document Constraints
Explain what the constraint implies:

```go
// Sum returns the sum of all elements.
// T must be a numeric type that supports addition.
func Sum[T Number](values []T) T
```

## Exercises

1. **Generic Stack**: Implement a thread-safe generic stack with Push, Pop, and Peek operations.

2. **Filter Function**: Write a generic Filter function:
   ```go
   func Filter[T any](slice []T, predicate func(T) bool) []T
   ```

3. **Result Type**: Create a generic Result type for error handling:
   ```go
   type Result[T any] struct {
       // Your implementation
   }
   ```

4. **Binary Tree**: Implement a generic binary search tree that works with any ordered type.

5. **Pipeline**: Create a generic pipeline that chains operations:
   ```go
   Pipeline([]int{1,2,3}).
       Map(func(x int) int { return x * 2 }).
       Filter(func(x int) bool { return x > 2 }).
       Result()  // [4, 6]
   ```

## Summary

Type parameters transform Go from a language where you choose between type safety and code reuse to one where you can have both. The design is quintessentially Go: powerful enough to solve real problems, simple enough to understand quickly, with just enough constraints to prevent footguns.

The key insights:
- Type parameters are like variables for types
- Constraints define what operations are available
- Type inference usually figures out what you mean
- Generic code compiles to efficient machine code
- Start concrete, generalize when patterns emerge

In the next chapter, we'll move from understanding generics to mastering them, exploring patterns and practices that have emerged from three years of real-world use.

---

*Continue to [Chapter 4: Generic Programming in Practice](chapter04.md)*