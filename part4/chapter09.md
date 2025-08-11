# Chapter 9: Concurrency Patterns Updated

## The Concurrency Renaissance

Go's concurrency story began with goroutines and channels, but it didn't end there. Modern Go provides sophisticated synchronization primitives, atomic types with methods, and battle-tested patterns that make concurrent code both safer and faster. This chapter explores how concurrency has evolved beyond "Don't communicate by sharing memory; share memory by communicating."

## Modern Sync Primitives

### sync.Map: The Concurrent Map

Before sync.Map, concurrent map access required manual locking:

```go
// The old way
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]interface{}
}

func (s *SafeMap) Get(key string) (interface{}, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    val, ok := s.m[key]
    return val, ok
}

// The modern way with sync.Map
var cache sync.Map

func modernMapUsage() {
    // Store
    cache.Store("key", "value")
    
    // Load
    if val, ok := cache.Load("key"); ok {
        fmt.Println(val)
    }
    
    // LoadOrStore
    actual, loaded := cache.LoadOrStore("key", "new-value")
    if loaded {
        fmt.Println("Already existed:", actual)
    }
    
    // LoadAndDelete
    if val, loaded := cache.LoadAndDelete("key"); loaded {
        fmt.Println("Deleted:", val)
    }
    
    // Range
    cache.Range(func(key, value interface{}) bool {
        fmt.Printf("%v: %v\n", key, value)
        return true  // Continue iteration
    })
    
    // Swap (Go 1.20+)
    old, loaded := cache.Swap("key", "newest-value")
    
    // CompareAndSwap (Go 1.20+)
    swapped := cache.CompareAndSwap("key", old, "even-newer")
}
```

When to use sync.Map:
- Many goroutines reading, few writing
- Keys are written once but read many times
- Disjoint sets of keys between goroutines

### sync.Pool: Object Recycling

Reduce GC pressure with object pools:

```go
// Define a pool for expensive objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        // Create new object when pool is empty
        return new(bytes.Buffer)
    },
}

func processWithPool(data []byte) {
    // Get from pool
    buf := bufferPool.Get().(*bytes.Buffer)
    
    // Return to pool when done
    defer func() {
        buf.Reset()  // Clean before returning
        bufferPool.Put(buf)
    }()
    
    // Use the buffer
    buf.Write(data)
    result := transform(buf.Bytes())
    return result
}

// Advanced pool pattern with type safety
type Pool[T any] struct {
    pool sync.Pool
}

func NewPool[T any](new func() T) *Pool[T] {
    return &Pool[T]{
        pool: sync.Pool{
            New: func() interface{} {
                return new()
            },
        },
    }
}

func (p *Pool[T]) Get() T {
    return p.pool.Get().(T)
}

func (p *Pool[T]) Put(v T) {
    p.pool.Put(v)
}

// Usage
var connPool = NewPool(func() *Connection {
    return &Connection{
        buffer: make([]byte, 4096),
    }
})
```

### sync.Once: Guaranteed Single Execution

Initialize resources exactly once:

```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{
            // Expensive initialization
            data: loadData(),
        }
    })
    return instance
}

// Advanced pattern: once with error handling
type OnceError struct {
    once sync.Once
    err  error
}

func (o *OnceError) Do(f func() error) error {
    o.once.Do(func() {
        o.err = f()
    })
    return o.err
}

// Usage
var initOnce OnceError

func Initialize() error {
    return initOnce.Do(func() error {
        // Initialization that might fail
        if err := connectDB(); err != nil {
            return err
        }
        return loadConfig()
    })
}
```

## Atomic Operations Evolution

### Typed Atomics (Go 1.19+)

Modern Go provides type-safe atomic operations:

```go
// Old way with atomic package functions
var counter int64

atomic.AddInt64(&counter, 1)
value := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 0)

// New way with atomic types
var counter atomic.Int64

counter.Add(1)
value := counter.Load()
counter.Store(0)
counter.CompareAndSwap(0, 1)

// All atomic types available
var (
    b  atomic.Bool
    i32 atomic.Int32
    i64 atomic.Int64
    u32 atomic.Uint32
    u64 atomic.Uint64
    ptr atomic.Pointer[*Node]
    val atomic.Value
)

// Atomic pointer with type safety
type Node struct {
    value int
    next  *Node
}

var head atomic.Pointer[*Node]

func push(value int) {
    newNode := &Node{value: value}
    for {
        oldHead := head.Load()
        newNode.next = oldHead
        if head.CompareAndSwap(oldHead, newNode) {
            break
        }
    }
}
```

### Atomic Patterns

Build lock-free data structures:

```go
// Lock-free counter with threshold
type ThresholdCounter struct {
    count     atomic.Int64
    threshold int64
    callback  func()
}

func (tc *ThresholdCounter) Increment() {
    new := tc.count.Add(1)
    if new == tc.threshold {
        go tc.callback()  // Trigger action at threshold
    }
}

// Atomic configuration updates
type Config struct {
    data atomic.Pointer[ConfigData]
}

type ConfigData struct {
    Timeout  time.Duration
    MaxConns int
    Features map[string]bool
}

func (c *Config) Update(new *ConfigData) {
    c.data.Store(new)
}

func (c *Config) Get() *ConfigData {
    return c.data.Load()
}

// Usage - no locks needed!
config := &Config{}
config.Update(&ConfigData{
    Timeout:  5 * time.Second,
    MaxConns: 100,
})

// Readers get consistent view
cfg := config.Get()
timeout := cfg.Timeout
```

## Error Group Pattern

The `golang.org/x/sync/errgroup` package revolutionized concurrent error handling:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, urls []string) ([]Result, error) {
    g, ctx := errgroup.WithContext(ctx)
    results := make([]Result, len(urls))
    
    for i, url := range urls {
        i, url := i, url  // Capture loop variables
        g.Go(func() error {
            result, err := fetch(ctx, url)
            if err != nil {
                return fmt.Errorf("fetching %s: %w", url, err)
            }
            results[i] = result
            return nil
        })
    }
    
    if err := g.Wait(); err != nil {
        return nil, err  // First error encountered
    }
    
    return results, nil
}

// With concurrency limit
func processWithLimit(ctx context.Context, items []Item) error {
    g, ctx := errgroup.WithContext(ctx)
    g.SetLimit(10)  // Max 10 concurrent goroutines
    
    for _, item := range items {
        item := item
        g.Go(func() error {
            return process(ctx, item)
        })
    }
    
    return g.Wait()
}

// Advanced: Pipeline with error group
func pipeline(ctx context.Context) error {
    g, ctx := errgroup.WithContext(ctx)
    
    // Stage 1: Generate
    numbers := make(chan int)
    g.Go(func() error {
        defer close(numbers)
        for i := 0; i < 100; i++ {
            select {
            case numbers <- i:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })
    
    // Stage 2: Square
    squared := make(chan int)
    g.Go(func() error {
        defer close(squared)
        for n := range numbers {
            select {
            case squared <- n * n:
            case <-ctx.Done():
                return ctx.Err()
            }
        }
        return nil
    })
    
    // Stage 3: Sum
    g.Go(func() error {
        sum := 0
        for n := range squared {
            sum += n
        }
        fmt.Println("Sum:", sum)
        return nil
    })
    
    return g.Wait()
}
```

## Semaphore Pattern

Weighted semaphores for resource limiting:

```go
import "golang.org/x/sync/semaphore"

type ConnectionPool struct {
    sem   *semaphore.Weighted
    conns chan *Connection
}

func NewConnectionPool(size int) *ConnectionPool {
    return &ConnectionPool{
        sem:   semaphore.NewWeighted(int64(size)),
        conns: make(chan *Connection, size),
    }
}

func (p *ConnectionPool) Acquire(ctx context.Context) (*Connection, error) {
    // Acquire semaphore
    if err := p.sem.Acquire(ctx, 1); err != nil {
        return nil, err
    }
    
    select {
    case conn := <-p.conns:
        return conn, nil
    default:
        // Create new connection
        return newConnection()
    }
}

func (p *ConnectionPool) Release(conn *Connection) {
    defer p.sem.Release(1)
    
    select {
    case p.conns <- conn:
        // Returned to pool
    default:
        // Pool full, close connection
        conn.Close()
    }
}

// Dynamic resource allocation
func processWithResources(ctx context.Context, jobs []Job) error {
    // Semaphore representing available memory (in MB)
    memSem := semaphore.NewWeighted(1024)  // 1GB total
    
    g, ctx := errgroup.WithContext(ctx)
    
    for _, job := range jobs {
        job := job
        g.Go(func() error {
            // Acquire memory for job
            memNeeded := int64(job.MemoryMB())
            if err := memSem.Acquire(ctx, memNeeded); err != nil {
                return err
            }
            defer memSem.Release(memNeeded)
            
            return job.Run(ctx)
        })
    }
    
    return g.Wait()
}
```

## Modern Channel Patterns

### Fan-Out/Fan-In with Generics

```go
func FanOut[T any](ctx context.Context, in <-chan T, workers int) []<-chan T {
    outs := make([]<-chan T, workers)
    
    for i := 0; i < workers; i++ {
        out := make(chan T)
        outs[i] = out
        
        go func() {
            defer close(out)
            for val := range in {
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }()
    }
    
    return outs
}

func FanIn[T any](ctx context.Context, inputs ...<-chan T) <-chan T {
    out := make(chan T)
    var wg sync.WaitGroup
    
    for _, in := range inputs {
        wg.Add(1)
        go func(ch <-chan T) {
            defer wg.Done()
            for val := range ch {
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
            }
        }(in)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Debounce and Throttle

Rate limiting patterns:

```go
// Debounce - only emit after quiet period
func Debounce[T any](ctx context.Context, in <-chan T, delay time.Duration) <-chan T {
    out := make(chan T)
    
    go func() {
        defer close(out)
        
        var timer *time.Timer
        var latest T
        var hasValue bool
        
        for {
            select {
            case val, ok := <-in:
                if !ok {
                    if timer != nil && hasValue {
                        <-timer.C
                        out <- latest
                    }
                    return
                }
                
                latest = val
                hasValue = true
                
                if timer != nil {
                    timer.Stop()
                }
                timer = time.NewTimer(delay)
                
            case <-timer.C:
                if hasValue {
                    out <- latest
                    hasValue = false
                }
                
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out
}

// Throttle - limit rate of emissions
func Throttle[T any](ctx context.Context, in <-chan T, limit time.Duration) <-chan T {
    out := make(chan T)
    
    go func() {
        defer close(out)
        
        ticker := time.NewTicker(limit)
        defer ticker.Stop()
        
        for {
            select {
            case val, ok := <-in:
                if !ok {
                    return
                }
                
                select {
                case out <- val:
                case <-ctx.Done():
                    return
                }
                
                <-ticker.C  // Wait for next tick
                
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return out
}
```

## Worker Pool Evolution

Modern worker pool with graceful shutdown:

```go
type Task func(context.Context) error

type WorkerPool struct {
    workers   int
    taskQueue chan Task
    wg        sync.WaitGroup
    once      sync.Once
    ctx       context.Context
    cancel    context.CancelFunc
}

func NewWorkerPool(workers int, queueSize int) *WorkerPool {
    ctx, cancel := context.WithCancel(context.Background())
    
    pool := &WorkerPool{
        workers:   workers,
        taskQueue: make(chan Task, queueSize),
        ctx:       ctx,
        cancel:    cancel,
    }
    
    pool.start()
    return pool
}

func (p *WorkerPool) start() {
    for i := 0; i < p.workers; i++ {
        p.wg.Add(1)
        go p.worker(i)
    }
}

func (p *WorkerPool) worker(id int) {
    defer p.wg.Done()
    
    for {
        select {
        case task, ok := <-p.taskQueue:
            if !ok {
                return  // Channel closed
            }
            
            if err := task(p.ctx); err != nil {
                log.Printf("Worker %d: task error: %v", id, err)
            }
            
        case <-p.ctx.Done():
            return
        }
    }
}

func (p *WorkerPool) Submit(task Task) error {
    select {
    case p.taskQueue <- task:
        return nil
    case <-p.ctx.Done():
        return fmt.Errorf("pool is shutting down")
    default:
        return fmt.Errorf("task queue is full")
    }
}

func (p *WorkerPool) Shutdown(ctx context.Context) error {
    p.once.Do(func() {
        close(p.taskQueue)
    })
    
    done := make(chan struct{})
    go func() {
        p.wg.Wait()
        close(done)
    }()
    
    select {
    case <-done:
        return nil
    case <-ctx.Done():
        p.cancel()  // Force shutdown
        return ctx.Err()
    }
}

// Advanced: Adaptive worker pool
type AdaptivePool struct {
    minWorkers int
    maxWorkers int
    current    atomic.Int32
    tasks      chan Task
    metrics    *Metrics
}

func (p *AdaptivePool) adjustWorkers() {
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        queueLen := len(p.tasks)
        current := p.current.Load()
        
        if queueLen > int(current)*2 && current < int32(p.maxWorkers) {
            // Spawn more workers
            p.spawnWorker()
            p.current.Add(1)
        } else if queueLen < int(current)/2 && current > int32(p.minWorkers) {
            // Signal worker to stop
            p.tasks <- nil  // Poison pill
            p.current.Add(-1)
        }
    }
}
```

## Pub-Sub Pattern

Type-safe publish-subscribe:

```go
type Event[T any] struct {
    Type string
    Data T
}

type PubSub[T any] struct {
    mu          sync.RWMutex
    subscribers map[string][]chan Event[T]
}

func NewPubSub[T any]() *PubSub[T] {
    return &PubSub[T]{
        subscribers: make(map[string][]chan Event[T]),
    }
}

func (ps *PubSub[T]) Subscribe(eventType string, buffer int) <-chan Event[T] {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    ch := make(chan Event[T], buffer)
    ps.subscribers[eventType] = append(ps.subscribers[eventType], ch)
    
    return ch
}

func (ps *PubSub[T]) Publish(event Event[T]) {
    ps.mu.RLock()
    subscribers := ps.subscribers[event.Type]
    ps.mu.RUnlock()
    
    for _, ch := range subscribers {
        select {
        case ch <- event:
        default:
            // Subscriber is slow, drop message
        }
    }
}

func (ps *PubSub[T]) Close() {
    ps.mu.Lock()
    defer ps.mu.Unlock()
    
    for _, subs := range ps.subscribers {
        for _, ch := range subs {
            close(ch)
        }
    }
    
    ps.subscribers = make(map[string][]chan Event[T])
}

// Usage
type UserEvent struct {
    UserID string
    Action string
}

func example() {
    ps := NewPubSub[UserEvent]()
    
    // Subscribe
    events := ps.Subscribe("user.login", 10)
    
    go func() {
        for event := range events {
            fmt.Printf("User %s logged in\n", event.Data.UserID)
        }
    }()
    
    // Publish
    ps.Publish(Event[UserEvent]{
        Type: "user.login",
        Data: UserEvent{UserID: "123", Action: "login"},
    })
}
```

## Circuit Breaker Pattern

Prevent cascading failures:

```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    maxFailures  int
    resetTimeout time.Duration
    
    mu           sync.Mutex
    state        State
    failures     int
    lastFailTime time.Time
    counts       atomic.Int64
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if !cb.canAttempt() {
        return fmt.Errorf("circuit breaker is open")
    }
    
    err := fn()
    cb.recordResult(err)
    
    return err
}

func (cb *CircuitBreaker) canAttempt() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    switch cb.state {
    case StateClosed:
        return true
        
    case StateOpen:
        if time.Since(cb.lastFailTime) > cb.resetTimeout {
            cb.state = StateHalfOpen
            cb.failures = 0
            return true
        }
        return false
        
    case StateHalfOpen:
        return true
        
    default:
        return false
    }
}

func (cb *CircuitBreaker) recordResult(err error) {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    cb.counts.Add(1)
    
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()
        
        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen
        }
    } else {
        if cb.state == StateHalfOpen {
            cb.state = StateClosed
        }
        cb.failures = 0
    }
}

// Advanced: Adaptive circuit breaker
type AdaptiveCircuitBreaker struct {
    window       *RollingWindow
    threshold    float64  // Error rate threshold
    minRequests  int
}

func (acb *AdaptiveCircuitBreaker) shouldOpen() bool {
    stats := acb.window.Stats()
    
    if stats.Total < acb.minRequests {
        return false
    }
    
    errorRate := float64(stats.Errors) / float64(stats.Total)
    return errorRate > acb.threshold
}
```

## Coordination Patterns

### Barrier Pattern

Synchronize multiple goroutines:

```go
type Barrier struct {
    n       int
    count   atomic.Int32
    ch      chan struct{}
    resetMu sync.Mutex
}

func NewBarrier(n int) *Barrier {
    return &Barrier{
        n:  n,
        ch: make(chan struct{}),
    }
}

func (b *Barrier) Wait() {
    if b.count.Add(1) == int32(b.n) {
        // Last one, release all
        close(b.ch)
        
        // Reset for next use
        b.resetMu.Lock()
        b.count.Store(0)
        b.ch = make(chan struct{})
        b.resetMu.Unlock()
    } else {
        // Wait for release
        <-b.ch
    }
}
```

### Phaser Pattern

Multi-phase synchronization:

```go
type Phaser struct {
    phase      atomic.Int32
    registered atomic.Int32
    arrived    atomic.Int32
    barrier    chan struct{}
}

func (p *Phaser) Register() {
    p.registered.Add(1)
}

func (p *Phaser) ArriveAndAwaitAdvance() {
    arrived := p.arrived.Add(1)
    registered := p.registered.Load()
    
    if arrived == registered {
        // Last to arrive, advance phase
        p.phase.Add(1)
        p.arrived.Store(0)
        
        // Signal all waiting
        close(p.barrier)
        p.barrier = make(chan struct{})
    } else {
        // Wait for phase advance
        <-p.barrier
    }
}
```

## Testing Concurrent Code

### Race Detection

```go
func TestConcurrentAccess(t *testing.T) {
    // Run with: go test -race
    
    var counter int
    var wg sync.WaitGroup
    
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++  // Race condition!
        }()
    }
    
    wg.Wait()
}
```

### Concurrent Test Helpers

```go
func TestConcurrentMap(t *testing.T) {
    m := NewConcurrentMap()
    
    // Parallel subtest
    t.Run("concurrent writes", func(t *testing.T) {
        t.Parallel()
        
        for i := 0; i < 100; i++ {
            i := i
            t.Run(fmt.Sprintf("writer-%d", i), func(t *testing.T) {
                t.Parallel()
                m.Set(fmt.Sprintf("key-%d", i), i)
            })
        }
    })
    
    // Verify all writes
    for i := 0; i < 100; i++ {
        val, ok := m.Get(fmt.Sprintf("key-%d", i))
        require.True(t, ok)
        require.Equal(t, i, val)
    }
}

// Stress testing
func TestStress(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping stress test")
    }
    
    const (
        workers = 100
        operations = 10000
    )
    
    start := time.Now()
    // Run stress test...
    duration := time.Since(start)
    
    ops := float64(workers*operations) / duration.Seconds()
    t.Logf("Performance: %.0f ops/sec", ops)
}
```

## Best Practices

### 1. Start Simple
```go
// Begin with channels and goroutines
// Add complexity only when needed
```

### 2. Explicit Cancellation
```go
// Always provide cancellation
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

### 3. Avoid Goroutine Leaks
```go
// Ensure goroutines can exit
go func() {
    defer cleanup()
    for {
        select {
        case <-done:
            return
        case work := <-ch:
            process(work)
        }
    }
}()
```

### 4. Test with -race
```bash
go test -race ./...
```

## Exercises

1. **Rate Limiter**: Implement a token bucket rate limiter using channels and goroutines.

2. **Priority Queue**: Build a concurrent priority queue with fair scheduling.

3. **MapReduce**: Create a generic MapReduce framework using modern patterns.

4. **Actor System**: Implement an actor model using channels and goroutines.

5. **Concurrent Cache**: Build an LRU cache safe for concurrent access.

## Summary

Go's concurrency primitives have evolved far beyond basic goroutines and channels. Modern patterns like error groups, typed atomics, and sophisticated synchronization primitives make concurrent Go code more robust and maintainable. The key is knowing when to use each pattern—channels for orchestration, atomics for simple state, locks for complex state, and higher-level patterns for coordination.

Key takeaways:
- Use sync.Map for read-heavy concurrent maps
- Error groups simplify concurrent error handling
- Typed atomics are safer and clearer
- Context should thread through all concurrent code
- Test with race detection always

Next, we'll explore how testing in Go has evolved with fuzzing, better benchmarks, and sophisticated test helpers.

---

*Continue to [Chapter 10: Advanced Testing](../part5/chapter10.md)*