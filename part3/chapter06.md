# Chapter 6: Context and Cancellation

## The Thread That Binds

Before context, Go programs leaked goroutines like a rusty bucket leaks water. Cancellation was ad-hoc, timeouts were manual, and passing request-scoped values required creative solutions. The context package, promoted to the standard library in Go 1.7, changed everything.

Today, context is the invisible thread that binds concurrent Go programs together, enabling graceful shutdowns, deadline propagation, and clean request tracing. If you see `ctx context.Context` as the first parameter of a function, you're looking at modern Go.

## Life Before Context

To appreciate context, remember the chaos it replaced:

```go
// The old way: channels for cancellation
func worker(done chan bool, work chan int, results chan int) {
    for {
        select {
        case <-done:
            return  // Hope we catch this in time
        case val := <-work:
            // Simulate work
            time.Sleep(time.Second)
            select {
            case results <- val * 2:
            case <-done:
                return  // Check again!
            }
        }
    }
}

// Manual timeout management
func withTimeout(fn func() error, timeout time.Duration) error {
    done := make(chan error, 1)
    go func() {
        done <- fn()
    }()
    
    select {
    case err := <-done:
        return err
    case <-time.After(timeout):
        // Function is still running! Can't stop it!
        return errors.New("timeout")
    }
}

// Request values through function parameters
func handleRequest(userID string, requestID string, traceID string, 
    timeout time.Duration, logger *log.Logger) {
    // Parameters proliferate through the call stack
    processUser(userID, requestID, traceID, timeout, logger)
}
```

## Understanding Context

Context carries deadlines, cancellation signals, and request-scoped values across API boundaries:

```go
// The context way
func worker(ctx context.Context, work chan int) chan int {
    results := make(chan int)
    
    go func() {
        defer close(results)
        for {
            select {
            case <-ctx.Done():
                // Context cancelled or timed out
                log.Printf("Worker stopped: %v", ctx.Err())
                return
            case val := <-work:
                // Simulate work
                select {
                case <-time.After(time.Second):
                    results <- val * 2
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    
    return results
}

func main() {
    // Create a context with timeout
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()  // Always call cancel!
    
    work := make(chan int)
    results := worker(ctx, work)
    
    // Send work
    go func() {
        for i := 0; i < 10; i++ {
            work <- i
        }
    }()
    
    // Collect results until timeout
    for result := range results {
        fmt.Println(result)
    }
}
```

## The Context Interface

The context interface is deceptively simple:

```go
type Context interface {
    // Deadline returns the time when work should be canceled
    Deadline() (deadline time.Time, ok bool)
    
    // Done returns a channel that's closed when work should be canceled
    Done() <-chan struct{}
    
    // Err returns nil if Done is not closed, or an error explaining why:
    // - Canceled if the context was canceled
    // - DeadlineExceeded if the deadline passed
    Err() error
    
    // Value returns the value associated with key, or nil
    Value(key any) any
}
```

## Creating Contexts

### The Root Contexts

```go
// Empty context that's never canceled
ctx := context.Background()  // Use at the top level

// Empty context for unclear situations (being phased out)
ctx := context.TODO()  // Use when unsure what context to use
```

### Deriving Contexts

Contexts form a tree, with each derived context inheriting from its parent:

```go
// With cancellation
ctx, cancel := context.WithCancel(parent)
defer cancel()  // Release resources

// With deadline
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(parent, deadline)
defer cancel()

// With timeout (convenience for deadline)
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

// With value
ctx := context.WithValue(parent, "user_id", "12345")
```

## Modern Context Features

### WithoutCancel (Go 1.21+)

Creates a context that's not canceled when parent is:

```go
func longRunningCleanup(ctx context.Context) {
    // Create a context that survives parent cancellation
    cleanupCtx := context.WithoutCancel(ctx)
    
    go func() {
        // This continues even if parent ctx is canceled
        <-ctx.Done()
        log.Println("Parent canceled, starting cleanup")
        
        // Use cleanupCtx for cleanup operations
        if err := doCleanup(cleanupCtx); err != nil {
            log.Printf("Cleanup failed: %v", err)
        }
    }()
}
```

### WithCancelCause (Go 1.20+)

Cancel with a specific error:

```go
ctx, cancel := context.WithCancelCause(parent)

// Cancel with reason
cancel(fmt.Errorf("user %s not authorized", userID))

// Check the cause
if err := context.Cause(ctx); err != nil {
    log.Printf("Context canceled: %v", err)
}

// Example: Cascading cancellation with reasons
func processRequest(ctx context.Context) error {
    ctx, cancel := context.WithCancelCause(ctx)
    defer cancel(nil)  // No error on success
    
    if err := authenticate(ctx); err != nil {
        cancel(fmt.Errorf("authentication failed: %w", err))
        return context.Cause(ctx)
    }
    
    if err := authorize(ctx); err != nil {
        cancel(fmt.Errorf("authorization failed: %w", err))
        return context.Cause(ctx)
    }
    
    return nil
}
```

### AfterFunc (Go 1.21+)

Run a function when context is done:

```go
func watchContext(ctx context.Context) {
    stop := context.AfterFunc(ctx, func() {
        log.Println("Context canceled, cleaning up resources")
        closeConnections()
        flushBuffers()
    })
    
    // Can prevent the function from running
    if shouldNotCleanup {
        stop()
    }
}

// Use for resource cleanup
func withResource(ctx context.Context) (*Resource, error) {
    r, err := acquireResource()
    if err != nil {
        return nil, err
    }
    
    // Automatically release when context is done
    context.AfterFunc(ctx, func() {
        r.Release()
    })
    
    return r, nil
}
```

## Context Patterns and Best Practices

### Pattern 1: Request Context

Standard pattern for HTTP handlers:

```go
type contextKey string

const (
    userIDKey     contextKey = "user_id"
    requestIDKey  contextKey = "request_id"
    traceIDKey    contextKey = "trace_id"
)

func middleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Add request ID
        requestID := r.Header.Get("X-Request-ID")
        if requestID == "" {
            requestID = generateRequestID()
        }
        
        ctx := context.WithValue(r.Context(), requestIDKey, requestID)
        
        // Add timeout
        ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
        defer cancel()
        
        // Add user ID after authentication
        userID := authenticateUser(r)
        ctx = context.WithValue(ctx, userIDKey, userID)
        
        // Pass enhanced context
        next.ServeHTTP(w, r.WithContext(ctx))
    }
}

func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Retrieve values
    userID, _ := ctx.Value(userIDKey).(string)
    requestID, _ := ctx.Value(requestIDKey).(string)
    
    // Use context for downstream calls
    user, err := getUserFromDB(ctx, userID)
    if err != nil {
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
            return
        }
        http.Error(w, "Internal error", http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}
```

### Pattern 2: Graceful Shutdown

Use context for coordinated shutdown:

```go
type Server struct {
    srv *http.Server
    db  *sql.DB
}

func (s *Server) Run(ctx context.Context) error {
    // Create error group with context
    g, ctx := errgroup.WithContext(ctx)
    
    // Start HTTP server
    g.Go(func() error {
        log.Println("Starting HTTP server")
        if err := s.srv.ListenAndServe(); err != http.ErrServerClosed {
            return fmt.Errorf("HTTP server failed: %w", err)
        }
        return nil
    })
    
    // Graceful shutdown on context cancellation
    g.Go(func() error {
        <-ctx.Done()
        log.Println("Shutting down HTTP server")
        
        shutdownCtx, cancel := context.WithTimeout(
            context.Background(), 10*time.Second)
        defer cancel()
        
        return s.srv.Shutdown(shutdownCtx)
    })
    
    // Background tasks
    g.Go(func() error {
        ticker := time.NewTicker(time.Minute)
        defer ticker.Stop()
        
        for {
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-ticker.C:
                if err := s.runMaintenance(ctx); err != nil {
                    log.Printf("Maintenance failed: %v", err)
                }
            }
        }
    })
    
    return g.Wait()
}

func main() {
    ctx, cancel := signal.NotifyContext(context.Background(), 
        os.Interrupt, syscall.SIGTERM)
    defer cancel()
    
    server := &Server{
        srv: &http.Server{Addr: ":8080"},
        db:  initDB(),
    }
    
    if err := server.Run(ctx); err != nil {
        log.Fatal(err)
    }
}
```

### Pattern 3: Fan-Out/Fan-In

Coordinate multiple concurrent operations:

```go
func fetchFromMultipleSources(ctx context.Context, query string) ([]Result, error) {
    sources := []string{"source1", "source2", "source3"}
    
    // Create a context with timeout for all fetches
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    type result struct {
        source string
        data   []Result
        err    error
    }
    
    resultCh := make(chan result, len(sources))
    
    // Fan-out: start a goroutine for each source
    for _, source := range sources {
        go func(src string) {
            data, err := fetchFromSource(ctx, src, query)
            resultCh <- result{source: src, data: data, err: err}
        }(source)
    }
    
    // Fan-in: collect results
    var allResults []Result
    var errors []error
    
    for i := 0; i < len(sources); i++ {
        select {
        case r := <-resultCh:
            if r.err != nil {
                errors = append(errors, 
                    fmt.Errorf("source %s: %w", r.source, r.err))
            } else {
                allResults = append(allResults, r.data...)
            }
        case <-ctx.Done():
            return nil, fmt.Errorf("fetch timeout: %w", ctx.Err())
        }
    }
    
    if len(errors) > 0 {
        return allResults, fmt.Errorf("partial failures: %w", 
            errors.Join(errors...))
    }
    
    return allResults, nil
}
```

### Pattern 4: Context-Aware Retry

Implement retry logic that respects context:

```go
func retryWithContext(ctx context.Context, fn func() error) error {
    backoff := time.Second
    maxBackoff := time.Minute
    
    for attempt := 1; ; attempt++ {
        // Check context before attempting
        if ctx.Err() != nil {
            return fmt.Errorf("context canceled before attempt %d: %w", 
                attempt, ctx.Err())
        }
        
        err := fn()
        if err == nil {
            return nil
        }
        
        // Check if error is retryable
        if !isRetryable(err) {
            return fmt.Errorf("non-retryable error: %w", err)
        }
        
        log.Printf("Attempt %d failed: %v, retrying in %v", 
            attempt, err, backoff)
        
        // Wait with context
        timer := time.NewTimer(backoff)
        select {
        case <-ctx.Done():
            timer.Stop()
            return fmt.Errorf("context canceled during retry: %w", ctx.Err())
        case <-timer.C:
            // Exponential backoff with jitter
            backoff = time.Duration(float64(backoff) * (1.5 + rand.Float64()*0.5))
            if backoff > maxBackoff {
                backoff = maxBackoff
            }
        }
    }
}
```

### Pattern 5: Distributed Tracing

Use context for trace propagation:

```go
type Span struct {
    TraceID  string
    SpanID   string
    ParentID string
    Start    time.Time
}

func StartSpan(ctx context.Context, name string) (context.Context, *Span) {
    parent, _ := ctx.Value("span").(*Span)
    
    span := &Span{
        SpanID:   generateID(),
        Start:    time.Now(),
    }
    
    if parent != nil {
        span.TraceID = parent.TraceID
        span.ParentID = parent.SpanID
    } else {
        span.TraceID = generateID()
    }
    
    log.Printf("[%s] Starting span %s: %s", span.TraceID, span.SpanID, name)
    
    return context.WithValue(ctx, "span", span), span
}

func (s *Span) End() {
    duration := time.Since(s.Start)
    log.Printf("[%s] Span %s completed in %v", s.TraceID, s.SpanID, duration)
}

func processOrder(ctx context.Context, orderID string) error {
    ctx, span := StartSpan(ctx, "processOrder")
    defer span.End()
    
    // Validate order
    ctx2, span2 := StartSpan(ctx, "validateOrder")
    err := validateOrder(ctx2, orderID)
    span2.End()
    if err != nil {
        return err
    }
    
    // Process payment
    ctx3, span3 := StartSpan(ctx, "processPayment")
    err = processPayment(ctx3, orderID)
    span3.End()
    if err != nil {
        return err
    }
    
    return nil
}
```

## Context Values: Use with Caution

### The Right Way

Context values should be request-scoped and cross API boundaries:

```go
// Good: request-scoped metadata
type RequestMeta struct {
    UserID    string
    RequestID string
    IPAddress string
}

type contextKey int

const requestMetaKey contextKey = iota

func WithRequestMeta(ctx context.Context, meta RequestMeta) context.Context {
    return context.WithValue(ctx, requestMetaKey, meta)
}

func GetRequestMeta(ctx context.Context) (RequestMeta, bool) {
    meta, ok := ctx.Value(requestMetaKey).(RequestMeta)
    return meta, ok
}
```

### The Wrong Way

Don't use context for optional parameters or dependency injection:

```go
// Bad: using context for configuration
func BadExample(ctx context.Context) {
    // Don't do this!
    db := ctx.Value("database").(*sql.DB)
    config := ctx.Value("config").(Config)
    logger := ctx.Value("logger").(*log.Logger)
}

// Good: explicit parameters
func GoodExample(ctx context.Context, db *sql.DB, config Config, logger *log.Logger) {
    // Clear dependencies
}
```

## Database Operations with Context

Modern database operations should respect context:

```go
func getUserByID(ctx context.Context, db *sql.DB, userID string) (*User, error) {
    // Query respects context timeout/cancellation
    row := db.QueryRowContext(ctx, 
        "SELECT id, name, email FROM users WHERE id = ?", userID)
    
    var user User
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %s not found", userID)
        }
        if ctx.Err() != nil {
            return nil, fmt.Errorf("query canceled: %w", ctx.Err())
        }
        return nil, fmt.Errorf("database error: %w", err)
    }
    
    return &user, nil
}

func bulkInsert(ctx context.Context, db *sql.DB, users []User) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return fmt.Errorf("begin transaction: %w", err)
    }
    defer tx.Rollback()
    
    stmt, err := tx.PrepareContext(ctx,
        "INSERT INTO users (id, name, email) VALUES (?, ?, ?)")
    if err != nil {
        return fmt.Errorf("prepare statement: %w", err)
    }
    defer stmt.Close()
    
    for _, user := range users {
        // Check context in long loops
        if ctx.Err() != nil {
            return fmt.Errorf("insert canceled: %w", ctx.Err())
        }
        
        _, err := stmt.ExecContext(ctx, user.ID, user.Name, user.Email)
        if err != nil {
            return fmt.Errorf("insert user %s: %w", user.ID, err)
        }
    }
    
    if err := tx.Commit(); err != nil {
        return fmt.Errorf("commit transaction: %w", err)
    }
    
    return nil
}
```

## HTTP Client with Context

Make HTTP requests that respect context:

```go
func fetchAPI(ctx context.Context, url string) (*APIResponse, error) {
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, fmt.Errorf("creating request: %w", err)
    }
    
    // Add headers from context
    if meta, ok := GetRequestMeta(ctx); ok {
        req.Header.Set("X-Request-ID", meta.RequestID)
        req.Header.Set("X-User-ID", meta.UserID)
    }
    
    client := &http.Client{
        Timeout: 10 * time.Second,  // Overall timeout
    }
    
    resp, err := client.Do(req)
    if err != nil {
        if ctx.Err() != nil {
            return nil, fmt.Errorf("request canceled: %w", ctx.Err())
        }
        return nil, fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("unexpected status: %d", resp.StatusCode)
    }
    
    var result APIResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decoding response: %w", err)
    }
    
    return &result, nil
}
```

## Testing with Context

### Testing Timeouts

```go
func TestTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    
    start := time.Now()
    err := slowOperation(ctx)
    duration := time.Since(start)
    
    if !errors.Is(err, context.DeadlineExceeded) {
        t.Errorf("expected deadline exceeded, got %v", err)
    }
    
    if duration > 150*time.Millisecond {
        t.Errorf("operation took too long: %v", duration)
    }
}
```

### Testing Cancellation

```go
func TestCancellation(t *testing.T) {
    ctx, cancel := context.WithCancel(context.Background())
    
    done := make(chan error, 1)
    go func() {
        done <- longRunningOperation(ctx)
    }()
    
    // Cancel after short delay
    time.Sleep(50 * time.Millisecond)
    cancel()
    
    select {
    case err := <-done:
        if !errors.Is(err, context.Canceled) {
            t.Errorf("expected canceled error, got %v", err)
        }
    case <-time.After(time.Second):
        t.Error("operation did not stop after cancellation")
    }
}
```

### Testing Context Values

```go
func TestContextValues(t *testing.T) {
    ctx := context.Background()
    ctx = context.WithValue(ctx, userIDKey, "test-user")
    ctx = context.WithValue(ctx, requestIDKey, "test-request")
    
    result, err := operationNeedingContext(ctx)
    if err != nil {
        t.Fatalf("operation failed: %v", err)
    }
    
    // Verify context values were used
    if !strings.Contains(result.AuditLog, "test-user") {
        t.Error("user ID not found in audit log")
    }
}
```

## Common Pitfalls

### Pitfall 1: Forgetting to Call Cancel

```go
// Bad: leaks resources
func bad() {
    ctx, _ := context.WithTimeout(context.Background(), time.Second)
    // Missing cancel!
    doWork(ctx)
}

// Good: always defer cancel
func good() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()  // Releases timer resources
    doWork(ctx)
}
```

### Pitfall 2: Storing Context in Structs

```go
// Bad: contexts shouldn't be stored
type Server struct {
    ctx context.Context  // Don't do this!
}

// Good: pass context to methods
type Server struct {
    // Other fields
}

func (s *Server) Handle(ctx context.Context, req Request) error {
    // Use the passed context
}
```

### Pitfall 3: Using Background Context Everywhere

```go
// Bad: ignoring parent context
func process(ctx context.Context, data []byte) error {
    // This ignores parent's deadline/cancellation!
    return saveToDatabase(context.Background(), data)
}

// Good: propagate context
func process(ctx context.Context, data []byte) error {
    return saveToDatabase(ctx, data)
}
```

## Best Practices

### 1. Context as First Parameter
```go
func DoSomething(ctx context.Context, arg1 string, arg2 int) error
```

### 2. Never Pass nil Context
```go
// If you don't have a context, use Background
if ctx == nil {
    ctx = context.Background()
}
```

### 3. Only Cancel Your Own Context
```go
// Don't cancel contexts you didn't create
func process(ctx context.Context) {
    // Create your own if you need to cancel
    myCtx, cancel := context.WithCancel(ctx)
    defer cancel()
}
```

### 4. Keep Context Values Minimal
```go
// Good: small, request-scoped data
ctx = context.WithValue(ctx, "request_id", requestID)

// Bad: large objects or application state
ctx = context.WithValue(ctx, "database", db)  // Don't!
```

## Exercises

1. **Timeout Cascade**: Implement a system where multiple operations share a deadline budget.

2. **Context Pool**: Create a worker pool that respects context cancellation.

3. **Circuit Breaker**: Build a context-aware circuit breaker that opens based on timeout patterns.

4. **Request Limiter**: Implement per-user request limiting using context values.

5. **Trace Aggregator**: Create a tracing system that aggregates timing data from context.

## Summary

Context has become the cornerstone of concurrent Go programs, providing a standard way to handle cancellation, deadlines, and request-scoped values. It transformed Go from a language where goroutine leaks were common to one where graceful shutdown is standard.

The key principles:
- Pass context as the first parameter
- Derive new contexts from existing ones
- Always call cancel functions
- Use values sparingly for request-scoped data
- Respect context cancellation throughout your code

Next, we'll explore another game-changing feature: embedding files directly into Go binaries, eliminating deployment complexities and enabling true single-binary distributions.

---

*Continue to [Chapter 7: Embedded Files and Resources](chapter07.md)*