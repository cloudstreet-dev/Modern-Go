# Chapter 8: Modern Go Performance

## Faster Without Trying

Here's something remarkable: Go programs written in 2015 run 20-40% faster today, without changing a single line of code. The compiler got smarter, the garbage collector got gentler, and the runtime got more efficient. This is the story of Go's performance evolution—how a language already known for speed became even faster.

## The Compiler Revolution

### Escape Analysis Evolution

The compiler's escape analysis determines what lives on the stack versus the heap:

```go
// Understanding escape analysis
func demonstrateEscape() {
    // Stack allocation - doesn't escape
    local := 42
    useValue(local)
    
    // Heap allocation - escapes via pointer
    escaped := 42
    usePointer(&escaped)  // Escapes to heap
    
    // Modern Go is smarter about this
    s := make([]int, 10)  // May stay on stack if doesn't escape
    processLocally(s)      // Compiler analyzes usage
}

// Check escape analysis
// $ go build -gcflags="-m=2" main.go
```

Modern escape analysis improvements:

```go
// Go 1.20+ better escape analysis
type Buffer struct {
    data [1024]byte
    len  int
}

func processBuffer() {
    // Old Go: might escape
    // New Go: stays on stack
    var buf Buffer
    fillBuffer(&buf)
    useBuffer(&buf)
}

// Slice backing arrays
func modernSlices() []byte {
    // Go 1.20+: better analysis of slice backing arrays
    buf := make([]byte, 0, 1024)
    return append(buf, "data"...)  // Smarter about capacity
}
```

### Inlining Improvements

Inlining eliminates function call overhead:

```go
// Modern Go inlines more aggressively
package main

// Go 1.20+ can inline functions with:
// - Loops (simple ones)
// - Type switches
// - Panic/recover (in some cases)

func calculate(x, y int) int {
    // This now gets inlined
    for i := 0; i < 3; i++ {
        x += y
    }
    return x
}

func process(data []int) int {
    sum := 0
    for _, v := range data {
        sum += calculate(v, 2)  // Inlined!
    }
    return sum
}

// Check inlining decisions
// $ go build -gcflags="-m=2 -l=4" main.go
```

### Profile-Guided Optimization (PGO)

Go 1.20+ introduced PGO for 2-14% performance improvements:

```go
// Step 1: Collect profile
// $ go test -cpuprofile=cpu.prof -run=^$ -bench=.

// Step 2: Build with PGO
// $ go build -pgo=cpu.prof

// Example: Hot path optimization
func hotPath(data []byte) int {
    // PGO identifies this as hot
    result := 0
    for _, b := range data {
        if b > 127 {  // PGO optimizes branch prediction
            result += processUnicode(b)
        } else {
            result += processASCII(b)
        }
    }
    return result
}

// PGO effects:
// - Better inlining decisions
// - Improved branch prediction
// - Devirtualization of interface calls
```

## Garbage Collection Evolution

### The Sub-Millisecond GC

Modern Go's GC pauses are typically under 500 microseconds:

```go
// Monitoring GC performance
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "time"
)

func monitorGC() {
    // Set GC percentage
    debug.SetGCPercent(100)  // Default
    
    // Get GC stats
    var stats runtime.MemStats
    runtime.ReadMemStats(&stats)
    
    fmt.Printf("GC runs: %d\n", stats.NumGC)
    fmt.Printf("Pause total: %v\n", time.Duration(stats.PauseTotalNs))
    fmt.Printf("Last pause: %v\n", time.Duration(stats.PauseNs[(stats.NumGC+255)%256]))
    
    // GODEBUG for detailed GC info
    // $ GODEBUG=gctrace=1 ./program
}

// GC-friendly patterns
type Node struct {
    value int
    next  *Node  // Pointer creates GC work
}

// Better: reduce pointers
type FlatNode struct {
    value int
    nextIndex int  // Index instead of pointer
}

type NodePool struct {
    nodes []FlatNode  // Single allocation
}
```

### Memory Ballast Technique

Control GC frequency with ballast:

```go
// Force less frequent GC for batch processing
func withBallast() {
    // Allocate large ballast
    ballast := make([]byte, 10<<30)  // 10GB ballast
    
    // GC will trigger less frequently
    runtime.KeepAlive(ballast)
    
    // Do memory-intensive work
    processBatch()
}

// Modern alternative: GOMEMLIMIT (Go 1.19+)
func withMemLimit() {
    // Set soft memory limit
    debug.SetMemoryLimit(10 << 30)  // 10GB limit
    
    // GC adapts to stay under limit
    processBatch()
}
```

## Runtime Optimizations

### Goroutine Scheduling

The scheduler has become more sophisticated:

```go
// Modern scheduler improvements
func schedulerDemo() {
    // Async preemption (Go 1.14+)
    // Goroutines can be preempted even in tight loops
    go func() {
        for {
            // Pre-1.14: Could monopolize CPU
            // Post-1.14: Gets preempted
            calculate()
        }
    }()
    
    // GOMAXPROCS tuning
    runtime.GOMAXPROCS(runtime.NumCPU())  // Default
    
    // Work stealing improvements
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // Scheduler distributes work efficiently
            process(id)
        }(i)
    }
    wg.Wait()
}

// CPU affinity patterns
func withAffinity() {
    // Lock goroutine to OS thread
    runtime.LockOSThread()
    defer runtime.UnlockOSThread()
    
    // Useful for:
    // - System calls that require thread-local state
    // - CPU-intensive work
    // - Reducing context switches
}
```

### Timer Improvements

Go 1.23 dramatically improved timer resolution on Windows:

```go
// Timer resolution improvements
func timerPrecision() {
    // Old Windows: ~15.6ms resolution
    // New Windows: ~0.5ms resolution
    
    start := time.Now()
    time.Sleep(1 * time.Millisecond)
    elapsed := time.Since(start)
    
    fmt.Printf("Sleep precision: %v\n", elapsed)
    
    // High-precision timing
    ticker := time.NewTicker(100 * time.Microsecond)
    defer ticker.Stop()
    
    for i := 0; i < 10; i++ {
        <-ticker.C
        // Much more accurate in modern Go
    }
}
```

## Memory Optimization Patterns

### Zero-Allocation Techniques

Write code that minimizes allocations:

```go
// String building without allocation
func buildString() string {
    // Bad: multiple allocations
    s := ""
    for i := 0; i < 100; i++ {
        s += "x"  // Allocates new string each time
    }
    
    // Good: single allocation
    var b strings.Builder
    b.Grow(100)  // Pre-allocate
    for i := 0; i < 100; i++ {
        b.WriteByte('x')
    }
    return b.String()
}

// Reuse allocations with sync.Pool
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processWithPool(data []byte) string {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    buf.Write(data)
    // Process...
    return buf.String()
}

// Stack-allocated arrays
func stackArrays() {
    // Stays on stack
    var array [1024]byte
    processArray(array[:])
    
    // Goes to heap
    slice := make([]byte, 1024)
    processSlice(slice)
}
```

### Struct Optimization

Optimize struct layout for memory and cache:

```go
// Before: poor alignment (40 bytes)
type PoorlyAligned struct {
    flag bool    // 1 byte + 7 padding
    id   int64   // 8 bytes
    name string  // 16 bytes (string header)
    age  int32   // 4 bytes + 4 padding
}

// After: better alignment (32 bytes)
type WellAligned struct {
    id   int64   // 8 bytes
    name string  // 16 bytes
    age  int32   // 4 bytes
    flag bool    // 1 byte + 3 padding
}

// Check struct size and alignment
func checkAlignment() {
    var p PoorlyAligned
    var w WellAligned
    
    fmt.Printf("Poorly aligned: %d bytes\n", unsafe.Sizeof(p))
    fmt.Printf("Well aligned: %d bytes\n", unsafe.Sizeof(w))
}

// Cache-friendly data structures
type CacheFriendly struct {
    // Hot fields together
    hotField1 int64
    hotField2 int64
    hotField3 int64
    
    _ [40]byte  // Padding to cache line
    
    // Cold fields together
    coldField1 string
    coldField2 time.Time
}
```

## Benchmarking and Profiling

### Modern Benchmarking

Write effective benchmarks:

```go
func BenchmarkModern(b *testing.B) {
    // Setup
    data := make([]int, 1000)
    for i := range data {
        data[i] = i
    }
    
    // Reset timer after setup
    b.ResetTimer()
    
    // Report allocations
    b.ReportAllocs()
    
    // The actual benchmark
    for i := 0; i < b.N; i++ {
        result := process(data)
        
        // Prevent compiler optimization
        runtime.KeepAlive(result)
    }
    
    // Report custom metrics
    b.ReportMetric(float64(len(data))/float64(b.Elapsed().Seconds()), "items/sec")
}

// Parallel benchmarks
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        // Each goroutine gets its own data
        data := make([]byte, 1024)
        
        for pb.Next() {
            processData(data)
        }
    })
}

// Sub-benchmarks for comparison
func BenchmarkAlgorithms(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            data := make([]int, size)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}
```

### Profiling in Production

Safe production profiling:

```go
import (
    "net/http"
    _ "net/http/pprof"
    "runtime"
)

func enableProfiling() {
    // CPU profiling endpoint
    http.HandleFunc("/debug/pprof/profile", func(w http.ResponseWriter, r *http.Request) {
        // Limit profiling duration
        seconds := 30
        if s := r.FormValue("seconds"); s != "" {
            if parsed, err := strconv.Atoi(s); err == nil && parsed < 60 {
                seconds = parsed
            }
        }
        
        pprof.StartCPUProfile(w)
        time.Sleep(time.Duration(seconds) * time.Second)
        pprof.StopCPUProfile()
    })
    
    // Continuous profiling with sampling
    runtime.SetCPUProfileRate(100)  // 100 Hz sampling
    
    // Memory profiling
    runtime.MemProfileRate = 1 << 20  // Sample every 1MB
}

// Profile analysis workflow
// $ go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
// $ go tool pprof -http=:8080 cpu.prof  # Web UI
```

## Concurrency Performance

### Channel Optimizations

Efficient channel patterns:

```go
// Buffered channels for performance
func efficientChannels() {
    // Size buffer to expected load
    ch := make(chan Task, runtime.NumCPU()*2)
    
    // Worker pool
    var wg sync.WaitGroup
    for i := 0; i < runtime.NumCPU(); i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for task := range ch {
                process(task)
            }
        }()
    }
    
    // Send work
    for _, task := range tasks {
        ch <- task
    }
    close(ch)
    wg.Wait()
}

// Select with default for non-blocking
func nonBlockingSend(ch chan<- int, value int) bool {
    select {
    case ch <- value:
        return true
    default:
        return false  // Channel full, don't block
    }
}

// Batching for efficiency
func batchProcessor(input <-chan Item) {
    batch := make([]Item, 0, 100)
    ticker := time.NewTicker(100 * time.Millisecond)
    defer ticker.Stop()
    
    for {
        select {
        case item, ok := <-input:
            if !ok {
                if len(batch) > 0 {
                    processBatch(batch)
                }
                return
            }
            batch = append(batch, item)
            
            if len(batch) >= 100 {
                processBatch(batch)
                batch = batch[:0]  // Reuse slice
            }
            
        case <-ticker.C:
            if len(batch) > 0 {
                processBatch(batch)
                batch = batch[:0]
            }
        }
    }
}
```

### Lock-Free Patterns

Reduce lock contention:

```go
// Sharded locks for concurrent maps
type ShardedMap struct {
    shards [256]shard
}

type shard struct {
    mu    sync.RWMutex
    items map[string]interface{}
}

func (m *ShardedMap) getShard(key string) *shard {
    hash := fnv32(key)
    return &m.shards[hash%256]
}

func (m *ShardedMap) Get(key string) (interface{}, bool) {
    shard := m.getShard(key)
    shard.mu.RLock()
    val, ok := shard.items[key]
    shard.mu.RUnlock()
    return val, ok
}

// Atomic operations instead of locks
type Counter struct {
    value atomic.Int64
}

func (c *Counter) Increment() int64 {
    return c.value.Add(1)
}

func (c *Counter) Get() int64 {
    return c.value.Load()
}

// Lock-free ring buffer
type RingBuffer struct {
    buffer []interface{}
    head   atomic.Uint64
    tail   atomic.Uint64
    mask   uint64
}

func NewRingBuffer(size int) *RingBuffer {
    // Size must be power of 2
    size = nextPowerOf2(size)
    return &RingBuffer{
        buffer: make([]interface{}, size),
        mask:   uint64(size - 1),
    }
}

func (r *RingBuffer) Push(item interface{}) bool {
    head := r.head.Load()
    tail := r.tail.Load()
    
    if head-tail >= uint64(len(r.buffer)) {
        return false  // Full
    }
    
    r.buffer[head&r.mask] = item
    r.head.Add(1)
    return true
}
```

## SIMD and Vectorization

Modern Go can use SIMD instructions:

```go
// Manual SIMD with assembly
//go:noescape
func addVectorAVX2(a, b, c []float32)

// Pure Go that may vectorize
func addVectors(a, b []float32) []float32 {
    c := make([]float32, len(a))
    
    // Compiler may vectorize this loop
    for i := range a {
        c[i] = a[i] + b[i]
    }
    
    return c
}

// Help compiler vectorize
func dotProduct(a, b []float32) float32 {
    if len(a) != len(b) {
        panic("length mismatch")
    }
    
    var sum float32
    
    // Process in chunks for better vectorization
    i := 0
    for ; i <= len(a)-4; i += 4 {
        sum += a[i]*b[i] + a[i+1]*b[i+1] + a[i+2]*b[i+2] + a[i+3]*b[i+3]
    }
    
    // Handle remaining elements
    for ; i < len(a); i++ {
        sum += a[i] * b[i]
    }
    
    return sum
}
```

## Compiler Directives

Control optimization with directives:

```go
// Prevent inlining
//go:noinline
func dontInline() {
    // Complex function we don't want inlined
}

// Force inlining (hint)
//go:inline
func alwaysInline() int {
    return 42
}

// Prevent escape to heap
//go:nosplit
func stackOnly() {
    // Function with limited stack check
}

// Bounds check elimination
func accessArray(arr []int, i int) int {
    // Manual bounds check elimination
    _ = arr[i]  // Single bounds check
    
    // These don't bounds check
    a := arr[i]
    b := arr[i]  // Compiler knows i is valid
    return a + b
}

// Branch prediction hints
func likely(b bool) bool {
    // Some architectures optimize for true
    return b
}

func process(data []byte) {
    for _, b := range data {
        if likely(b < 128) {  // ASCII most common
            processASCII(b)
        } else {
            processUnicode(b)
        }
    }
}
```

## Platform-Specific Optimizations

### Linux Optimizations

```go
//go:build linux

package perf

import (
    "syscall"
    "golang.org/x/sys/unix"
)

// Use mmap for large files
func mmapFile(path string) ([]byte, error) {
    file, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer file.Close()
    
    stat, _ := file.Stat()
    size := stat.Size()
    
    data, err := unix.Mmap(int(file.Fd()), 0, int(size),
        unix.PROT_READ, unix.MAP_PRIVATE)
    
    return data, err
}

// TCP optimizations
func optimizeTCP(conn *net.TCPConn) {
    conn.SetNoDelay(true)  // Disable Nagle
    conn.SetKeepAlive(true)
    conn.SetKeepAlivePeriod(30 * time.Second)
    
    // Set socket buffer sizes
    if file, err := conn.File(); err == nil {
        syscall.SetsockoptInt(int(file.Fd()), 
            syscall.SOL_SOCKET, syscall.SO_RCVBUF, 1<<20)
    }
}
```

### Architecture-Specific

```go
//go:build amd64

package arch

import "math/bits"

// Use CPU intrinsics
func countBits(x uint64) int {
    return bits.OnesCount64(x)  // Uses POPCNT on amd64
}

// Cache line aware
const CacheLineSize = 64

type PaddedCounter struct {
    value uint64
    _     [CacheLineSize - 8]byte  // Pad to cache line
}
```

## Real-World Optimization Example

Let's optimize a real service:

```go
// Before optimization
type SlowService struct {
    mu    sync.Mutex
    cache map[string]*Result
}

func (s *SlowService) Process(key string) (*Result, error) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    if r, ok := s.cache[key]; ok {
        return r, nil
    }
    
    // Expensive computation
    data := fetchData(key)
    result := compute(data)
    
    s.cache[key] = result
    return result, nil
}

// After optimization
type FastService struct {
    cache sync.Map  // Lock-free for reads
    pool  sync.Pool // Reuse allocations
}

func (s *FastService) Process(key string) (*Result, error) {
    // Fast path: lock-free read
    if v, ok := s.cache.Load(key); ok {
        return v.(*Result), nil
    }
    
    // Get buffer from pool
    buf := s.pool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        s.pool.Put(buf)
    }()
    
    // Compute with reused buffer
    result := computeWithBuffer(key, buf)
    
    // Store (may race, but idempotent)
    actual, _ := s.cache.LoadOrStore(key, result)
    return actual.(*Result), nil
}

// Benchmark results:
// BenchmarkSlowService-8    100000    15234 ns/op    4096 B/op    52 allocs/op
// BenchmarkFastService-8   1000000     1052 ns/op     128 B/op     2 allocs/op
```

## Best Practices

### 1. Measure First
```go
// Always benchmark before optimizing
func BenchmarkBefore(b *testing.B) {
    // Establish baseline
}
```

### 2. Optimize Hot Paths
```go
// Focus on code that runs frequently
if isHotPath {
    // Optimize here
} else {
    // Clarity over performance
}
```

### 3. Reduce Allocations
```go
// Reuse instead of allocate
var buffer [1024]byte  // Stack allocated
// vs
buffer := make([]byte, 1024)  // Heap allocated
```

### 4. Batch Operations
```go
// Process in batches
batch := make([]Item, 0, 100)
// Process when full or timeout
```

## Exercises

1. **Profile and Optimize**: Take a slow function and use profiling to identify and fix bottlenecks.

2. **Zero-Allocation Server**: Build an HTTP handler that processes requests without heap allocations.

3. **Lock-Free Queue**: Implement a high-performance lock-free queue.

4. **SIMD Optimization**: Write vector operations that the compiler can optimize.

5. **Memory Pool**: Create an efficient memory pool for a specific data structure.

## Summary

Modern Go's performance improvements are remarkable. Through compiler enhancements, GC refinements, and runtime optimizations, Go programs get faster with each release. The key insight: performance isn't just about writing fast code—it's about understanding how the compiler, runtime, and hardware work together.

Key takeaways:
- Profile before optimizing
- Reduce allocations for big wins
- Use PGO for free performance
- Understand escape analysis
- Leverage modern runtime improvements

Next, we'll explore how concurrency patterns have evolved with new synchronization primitives and patterns that make concurrent Go code both faster and safer.

---

*Continue to [Chapter 9: Concurrency Patterns Updated](chapter09.md)*