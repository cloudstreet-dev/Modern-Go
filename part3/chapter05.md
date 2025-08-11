# Chapter 5: Error Handling Evolution

## Beyond "if err != nil"

Go's error handling has been both its most mocked feature and its secret strength. While other languages hide errors behind exceptions, Go makes them explicit, unavoidable, and—since 2019—surprisingly sophisticated.

The evolution from simple error values to wrapped errors, error trees, and sophisticated error inspection has transformed error handling from a chore into a powerful debugging and observability tool.

## The Pre-2019 Era

Before Go 1.13, error handling was simple but limited:

```go
// The old way: errors as strings
err := errors.New("something went wrong")

// Or formatted errors
err := fmt.Errorf("failed to process item %d", itemID)

// Custom error types for more info
type ValidationError struct {
    Field string
    Value interface{}
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s: invalid value %v", 
        e.Field, e.Value)
}

// Checking errors was limited
if err != nil {
    // Is it a ValidationError? Need type assertion
    if valErr, ok := err.(ValidationError); ok {
        // Handle validation error
        log.Printf("Field %s failed validation", valErr.Field)
    }
}
```

The problems were clear:
- No error context or wrapping
- Difficult to check error types through layers
- Lost stack traces
- Manual error decoration

## Error Wrapping: The Game Changer

Go 1.13 introduced error wrapping with the `%w` verb:

```go
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        // Wrap the error with context
        return nil, fmt.Errorf("reading config file %s: %w", path, err)
    }
    
    var cfg Config
    if err := json.Unmarshal(data, &cfg); err != nil {
        // Another layer of wrapping
        return nil, fmt.Errorf("parsing config file %s: %w", path, err)
    }
    
    return &cfg, nil
}

func startServer() error {
    cfg, err := readConfig("/etc/app/config.json")
    if err != nil {
        // Wrap again at each level
        return fmt.Errorf("server startup failed: %w", err)
    }
    // ...
}

// The final error contains the full context chain:
// "server startup failed: reading config file /etc/app/config.json: open /etc/app/config.json: no such file or directory"
```

## errors.Is: Smart Error Checking

`errors.Is` checks if an error matches a target, even through wrapping:

```go
var (
    ErrNotFound   = errors.New("not found")
    ErrPermission = errors.New("permission denied")
    ErrTimeout    = errors.New("operation timed out")
)

func fetchUser(id string) (*User, error) {
    user, err := db.GetUser(id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("user %s: %w", id, ErrNotFound)
        }
        return nil, fmt.Errorf("fetching user %s: %w", id, err)
    }
    return user, nil
}

func handleRequest(userID string) {
    user, err := fetchUser(userID)
    if err != nil {
        // Works even though ErrNotFound is wrapped!
        if errors.Is(err, ErrNotFound) {
            http.Error(w, "User not found", 404)
            return
        }
        if errors.Is(err, context.DeadlineExceeded) {
            http.Error(w, "Request timeout", 504)
            return
        }
        // Generic error
        http.Error(w, "Internal error", 500)
        return
    }
    // ...
}
```

### Custom Is Support

Types can implement their own `Is` method:

```go
type APIError struct {
    Code    string
    Message string
    Status  int
}

func (e APIError) Error() string {
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e APIError) Is(target error) bool {
    t, ok := target.(APIError)
    if !ok {
        return false
    }
    // Match if codes are the same (ignore message/status)
    return e.Code == t.Code
}

var ErrRateLimit = APIError{Code: "RATE_LIMIT"}

func callAPI() error {
    // Specific instance with full details
    return fmt.Errorf("API call failed: %w", APIError{
        Code:    "RATE_LIMIT",
        Message: "Too many requests",
        Status:  429,
    })
}

// Check matches despite different message/status
if errors.Is(err, ErrRateLimit) {
    // Handle rate limiting
}
```

## errors.As: Type Extraction

`errors.As` finds and extracts specific error types from the chain:

```go
type NetworkError struct {
    Op       string
    Net      string
    Addr     string
    Timeout  bool
    Temporary bool
}

func (e NetworkError) Error() string {
    return fmt.Sprintf("network error during %s on %s://%s (timeout=%v)", 
        e.Op, e.Net, e.Addr, e.Timeout)
}

func (e NetworkError) IsTimeout() bool { return e.Timeout }
func (e NetworkError) IsTemporary() bool { return e.Temporary }

func makeRequest(url string) error {
    // Simulate network error
    netErr := NetworkError{
        Op:       "dial",
        Net:      "tcp",
        Addr:     "example.com:443",
        Timeout:  true,
        Temporary: true,
    }
    return fmt.Errorf("request to %s failed: %w", url, netErr)
}

func robustRequest(url string) error {
    err := makeRequest(url)
    if err != nil {
        var netErr NetworkError
        // Extract NetworkError from wrapped error
        if errors.As(err, &netErr) {
            if netErr.IsTemporary() {
                // Retry temporary errors
                log.Printf("Temporary error, retrying: %v", netErr)
                return makeRequest(url)
            }
            if netErr.IsTimeout() {
                return fmt.Errorf("request timed out: %w", err)
            }
        }
        return err
    }
    return nil
}
```

### Interface Extraction

`errors.As` works with interfaces too:

```go
type Retryable interface {
    error
    IsRetryable() bool
}

func processWithRetry(fn func() error, maxRetries int) error {
    var lastErr error
    
    for i := 0; i < maxRetries; i++ {
        if err := fn(); err != nil {
            lastErr = err
            
            var retryable Retryable
            if errors.As(err, &retryable) && retryable.IsRetryable() {
                log.Printf("Attempt %d failed (retryable): %v", i+1, err)
                time.Sleep(time.Second * time.Duration(i+1))
                continue
            }
            
            // Non-retryable error
            return err
        }
        
        return nil  // Success
    }
    
    return fmt.Errorf("failed after %d retries: %w", maxRetries, lastErr)
}
```

## errors.Join: Multiple Errors (Go 1.20+)

Go 1.20 introduced `errors.Join` for combining multiple errors:

```go
func validateUser(u User) error {
    var errs []error
    
    if u.Name == "" {
        errs = append(errs, errors.New("name is required"))
    }
    
    if u.Age < 0 || u.Age > 150 {
        errs = append(errs, fmt.Errorf("invalid age: %d", u.Age))
    }
    
    if u.Email != "" && !isValidEmail(u.Email) {
        errs = append(errs, fmt.Errorf("invalid email: %s", u.Email))
    }
    
    // Join all errors into one
    return errors.Join(errs...)
}

// errors.Join creates a tree structure
err := validateUser(User{Name: "", Age: 200, Email: "bad"})
// err.Error() returns multi-line output:
// name is required
// invalid age: 200
// invalid email: bad

// Is and As work on all joined errors
if errors.Is(err, ErrInvalidEmail) {
    // True if any joined error matches
}
```

### Working with Joined Errors

```go
type ValidationErrors struct {
    Errors []error
}

func (e ValidationErrors) Error() string {
    var msgs []string
    for _, err := range e.Errors {
        msgs = append(msgs, err.Error())
    }
    return strings.Join(msgs, "; ")
}

// Implement Unwrap to support Is/As
func (e ValidationErrors) Unwrap() []error {
    return e.Errors
}

func validateOrder(order Order) error {
    var errs ValidationErrors
    
    if order.Total <= 0 {
        errs.Errors = append(errs.Errors, 
            fmt.Errorf("invalid total: %v", order.Total))
    }
    
    for i, item := range order.Items {
        if err := validateItem(item); err != nil {
            errs.Errors = append(errs.Errors,
                fmt.Errorf("item %d: %w", i, err))
        }
    }
    
    if len(errs.Errors) > 0 {
        return errs
    }
    return nil
}
```

## Modern Error Patterns

### Pattern 1: Structured Errors

Create rich error types with structured data:

```go
type AppError struct {
    Op         string                 // Operation
    Kind       ErrorKind              // Category
    Err        error                  // Wrapped error
    StatusCode int                    // HTTP status
    Context    map[string]interface{} // Additional context
    Stack      []byte                 // Stack trace
}

type ErrorKind int

const (
    KindInternal ErrorKind = iota
    KindNotFound
    KindValidation
    KindPermission
    KindRateLimit
    KindTimeout
)

func (e *AppError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %v", e.Op, e.Err)
    }
    return fmt.Sprintf("%s: %v", e.Op, e.Kind)
}

func (e *AppError) Unwrap() error {
    return e.Err
}

func (e *AppError) Is(target error) bool {
    if target, ok := target.(*AppError); ok {
        return e.Kind == target.Kind
    }
    return false
}

// Constructor with stack trace
func NewAppError(op string, kind ErrorKind, err error) *AppError {
    return &AppError{
        Op:      op,
        Kind:    kind,
        Err:     err,
        Stack:   debug.Stack(),
        Context: make(map[string]interface{}),
    }
}

// Fluent API for adding context
func (e *AppError) WithContext(key string, value interface{}) *AppError {
    e.Context[key] = value
    return e
}

func (e *AppError) WithStatusCode(code int) *AppError {
    e.StatusCode = code
    return e
}

// Usage
func getUserByID(id string) (*User, error) {
    user, err := db.Query("SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, NewAppError("getUserByID", KindNotFound, err).
                WithContext("user_id", id).
                WithStatusCode(404)
        }
        return nil, NewAppError("getUserByID", KindInternal, err).
            WithContext("user_id", id).
            WithContext("query", "SELECT * FROM users").
            WithStatusCode(500)
    }
    return user, nil
}
```

### Pattern 2: Error Aggregation

Collect and handle multiple errors:

```go
type ErrorCollector struct {
    mu     sync.Mutex
    errors []error
}

func (c *ErrorCollector) Add(err error) {
    if err == nil {
        return
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    c.errors = append(c.errors, err)
}

func (c *ErrorCollector) Err() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if len(c.errors) == 0 {
        return nil
    }
    
    return errors.Join(c.errors...)
}

func (c *ErrorCollector) HasErrors() bool {
    c.mu.Lock()
    defer c.mu.Unlock()
    return len(c.errors) > 0
}

// Usage in concurrent operations
func processItems(items []Item) error {
    var wg sync.WaitGroup
    collector := &ErrorCollector{}
    
    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := processItem(item); err != nil {
                collector.Add(fmt.Errorf("item %s: %w", item.ID, err))
            }
        }(item)
    }
    
    wg.Wait()
    return collector.Err()
}
```

### Pattern 3: Error Recovery and Cleanup

Ensure cleanup even with errors:

```go
type Cleanup struct {
    funcs []func() error
    mu    sync.Mutex
}

func (c *Cleanup) Add(fn func() error) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.funcs = append(c.funcs, fn)
}

func (c *Cleanup) Run() error {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    var errs []error
    // Run in reverse order (LIFO)
    for i := len(c.funcs) - 1; i >= 0; i-- {
        if err := c.funcs[i](); err != nil {
            errs = append(errs, err)
        }
    }
    
    return errors.Join(errs...)
}

func complexOperation() (err error) {
    cleanup := &Cleanup{}
    defer func() {
        if cleanupErr := cleanup.Run(); cleanupErr != nil {
            err = errors.Join(err, fmt.Errorf("cleanup failed: %w", cleanupErr))
        }
    }()
    
    // Open resource 1
    r1, err := openResource1()
    if err != nil {
        return fmt.Errorf("opening resource1: %w", err)
    }
    cleanup.Add(r1.Close)
    
    // Open resource 2
    r2, err := openResource2()
    if err != nil {
        return fmt.Errorf("opening resource2: %w", err)
    }
    cleanup.Add(r2.Close)
    
    // Do work...
    if err := doWork(r1, r2); err != nil {
        return fmt.Errorf("work failed: %w", err)
    }
    
    return nil
}
```

### Pattern 4: Sentinel Errors with Context

Enhance sentinel errors with additional context:

```go
var (
    ErrDatabase = errors.New("database error")
    ErrNetwork  = errors.New("network error")
    ErrAuth     = errors.New("authentication error")
)

type SentinelWrapper struct {
    sentinel error
    msg      string
    fields   map[string]interface{}
}

func (w SentinelWrapper) Error() string {
    return fmt.Sprintf("%s: %s", w.msg, w.sentinel)
}

func (w SentinelWrapper) Unwrap() error {
    return w.sentinel
}

func Wrap(sentinel error, msg string) SentinelWrapper {
    return SentinelWrapper{
        sentinel: sentinel,
        msg:      msg,
        fields:   make(map[string]interface{}),
    }
}

func (w SentinelWrapper) With(key string, value interface{}) SentinelWrapper {
    w.fields[key] = value
    return w
}

// Usage
func authenticate(token string) error {
    if token == "" {
        return Wrap(ErrAuth, "missing token").
            With("timestamp", time.Now()).
            With("ip", getClientIP())
    }
    
    if !isValidToken(token) {
        return Wrap(ErrAuth, "invalid token").
            With("token_prefix", token[:4]).
            With("timestamp", time.Now())
    }
    
    return nil
}

// Check still works with wrapped sentinel
if errors.Is(err, ErrAuth) {
    // Handle auth error
}
```

## Error Formatting and Presentation

### Custom Error Formatting

Control how errors are displayed:

```go
type FormattedError struct {
    error
    format string
    args   []interface{}
}

func (e FormattedError) Error() string {
    return fmt.Sprintf(e.format, e.args...)
}

func (e FormattedError) Format(f fmt.State, verb rune) {
    switch verb {
    case 'v':
        if f.Flag('+') {
            // Verbose format with details
            fmt.Fprintf(f, "%s\n", e.Error())
            if e.error != nil {
                fmt.Fprintf(f, "Caused by: %+v", e.error)
            }
        } else {
            fmt.Fprintf(f, "%s", e.Error())
        }
    case 's':
        fmt.Fprintf(f, "%s", e.Error())
    case 'q':
        fmt.Fprintf(f, "%q", e.Error())
    }
}

// Usage
err := FormattedError{
    error:  io.EOF,
    format: "unexpected end of file while reading %s at line %d",
    args:   []interface{}{"config.json", 42},
}

fmt.Printf("%v\n", err)   // Simple format
fmt.Printf("%+v\n", err)  // Verbose format with cause
```

### Error Logging

Structure errors for better logging:

```go
type LoggableError interface {
    error
    LogFields() map[string]interface{}
}

type BusinessError struct {
    Code      string
    Message   string
    UserID    string
    RequestID string
    Timestamp time.Time
    cause     error
}

func (e BusinessError) Error() string {
    if e.cause != nil {
        return fmt.Sprintf("[%s] %s: %v", e.Code, e.Message, e.cause)
    }
    return fmt.Sprintf("[%s] %s", e.Code, e.Message)
}

func (e BusinessError) Unwrap() error {
    return e.cause
}

func (e BusinessError) LogFields() map[string]interface{} {
    return map[string]interface{}{
        "error_code": e.Code,
        "user_id":    e.UserID,
        "request_id": e.RequestID,
        "timestamp":  e.Timestamp,
        "message":    e.Message,
    }
}

// Integration with structured logging
func logError(err error) {
    if loggable, ok := err.(LoggableError); ok {
        slog.Error("business error", 
            slog.Any("fields", loggable.LogFields()),
            slog.String("error", err.Error()))
    } else {
        slog.Error("error occurred", slog.String("error", err.Error()))
    }
}
```

## Testing Errors

### Testing Error Conditions

```go
func TestErrorHandling(t *testing.T) {
    tests := []struct {
        name      string
        input     string
        wantErr   error
        wantMsg   string
        checkType bool
    }{
        {
            name:    "empty input",
            input:   "",
            wantErr: ErrInvalidInput,
        },
        {
            name:    "too long",
            input:   strings.Repeat("a", 1001),
            wantErr: ErrTooLong,
            wantMsg: "exceeds maximum length",
        },
        {
            name:      "network timeout",
            input:     "timeout-trigger",
            checkType: true,  // Check for NetworkError type
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := process(tt.input)
            
            if tt.wantErr != nil {
                if !errors.Is(err, tt.wantErr) {
                    t.Errorf("want error %v, got %v", tt.wantErr, err)
                }
            }
            
            if tt.wantMsg != "" {
                if !strings.Contains(err.Error(), tt.wantMsg) {
                    t.Errorf("error message should contain %q, got %q", 
                        tt.wantMsg, err.Error())
                }
            }
            
            if tt.checkType {
                var netErr NetworkError
                if !errors.As(err, &netErr) {
                    t.Errorf("expected NetworkError type, got %T", err)
                }
            }
        })
    }
}
```

### Error Injection for Testing

```go
type ErrorInjector struct {
    errors map[string]error
    mu     sync.RWMutex
}

func (i *ErrorInjector) SetError(op string, err error) {
    i.mu.Lock()
    defer i.mu.Unlock()
    if i.errors == nil {
        i.errors = make(map[string]error)
    }
    i.errors[op] = err
}

func (i *ErrorInjector) CheckError(op string) error {
    i.mu.RLock()
    defer i.mu.RUnlock()
    return i.errors[op]
}

// In production code
var testInjector *ErrorInjector  // nil in production

func riskyOperation() error {
    // Allow error injection in tests
    if testInjector != nil {
        if err := testInjector.CheckError("riskyOperation"); err != nil {
            return err
        }
    }
    
    // Normal operation
    return nil
}

// In tests
func TestErrorScenarios(t *testing.T) {
    injector := &ErrorInjector{}
    testInjector = injector
    defer func() { testInjector = nil }()
    
    // Inject specific error
    injector.SetError("riskyOperation", ErrTimeout)
    
    err := riskyOperation()
    if !errors.Is(err, ErrTimeout) {
        t.Errorf("expected timeout error, got %v", err)
    }
}
```

## Performance Considerations

### Error Allocation

Be mindful of error allocations in hot paths:

```go
// Avoid creating errors in hot paths
func validateFast(n int) error {
    if n < 0 {
        return errors.New("negative number")  // Allocates every time
    }
    return nil
}

// Better: reuse sentinel errors
var ErrNegativeNumber = errors.New("negative number")

func validateFaster(n int) error {
    if n < 0 {
        return ErrNegativeNumber  // No allocation
    }
    return nil
}

// For errors with context, consider pooling
var errorPool = sync.Pool{
    New: func() interface{} {
        return &ValueError{}
    },
}

type ValueError struct {
    Field string
    Value interface{}
}

func (e *ValueError) Error() string {
    return fmt.Sprintf("invalid value %v for field %s", e.Value, e.Field)
}

func (e *ValueError) Reset() {
    e.Field = ""
    e.Value = nil
}

func validateWithPool(field string, value interface{}) error {
    if !isValid(value) {
        err := errorPool.Get().(*ValueError)
        err.Field = field
        err.Value = value
        return err
    }
    return nil
}

// Return to pool when done
func handleError(err error) {
    if valErr, ok := err.(*ValueError); ok {
        defer func() {
            valErr.Reset()
            errorPool.Put(valErr)
        }()
        // Handle error
    }
}
```

## Best Practices

### 1. Wrap Errors at System Boundaries
```go
func (s *Service) GetUser(ctx context.Context, id string) (*User, error) {
    user, err := s.db.QueryUser(ctx, id)
    if err != nil {
        // Add context at service boundary
        return nil, fmt.Errorf("service.GetUser(%s): %w", id, err)
    }
    return user, nil
}
```

### 2. Use Sentinel Errors for API Contracts
```go
package myapi

var (
    ErrNotFound   = errors.New("myapi: not found")
    ErrUnauthorized = errors.New("myapi: unauthorized")
    ErrRateLimit  = errors.New("myapi: rate limit exceeded")
)

// Clients can reliably check these
if errors.Is(err, myapi.ErrNotFound) {
    // Handle not found
}
```

### 3. Include Actionable Information
```go
// Bad
return errors.New("operation failed")

// Good
return fmt.Errorf("failed to connect to database %s:%d (timeout=%v): %w", 
    host, port, timeout, err)
```

### 4. Don't Wrap Twice
```go
// Bad: double wrapping
err := doSomething()
if err != nil {
    return fmt.Errorf("failed: %w", fmt.Errorf("doSomething: %w", err))
}

// Good: single wrap with context
err := doSomething()
if err != nil {
    return fmt.Errorf("operation failed in doSomething: %w", err)
}
```

## Exercises

1. **Custom Error Chain**: Implement an error type that maintains a full chain of operations with timestamps.

2. **Retry Policy**: Create an error-based retry policy that examines error types to determine retry strategy.

3. **Error Reporter**: Build an error reporting system that extracts structured data from errors for monitoring.

4. **Circuit Breaker**: Implement a circuit breaker that uses error patterns to determine when to open.

5. **Error Recovery**: Design a system that can recover from specific errors by taking corrective actions.

## Summary

Go's error handling has evolved from simple value checking to a sophisticated system for error context, wrapping, and inspection. The key improvements—`%w` formatting, `errors.Is`, `errors.As`, and `errors.Join`—transform errors from mere failure indicators into rich debugging and observability tools.

Modern Go errors are:
- **Contextual**: Each layer adds meaningful context
- **Inspectable**: Types and values can be extracted from wrapped errors
- **Composable**: Multiple errors can be combined
- **Testable**: Error conditions can be precisely checked

Next, we'll explore another foundational addition to modern Go: the context package and its role in cancellation, deadlines, and request-scoped values.

---

*Continue to [Chapter 6: Context and Cancellation](chapter06.md)*