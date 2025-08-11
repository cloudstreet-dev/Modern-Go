# Appendix B: Quick Reference

## Modern Go Syntax Cheat Sheet

### Generics (Go 1.18+)

```go
// Type parameters
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Type constraints
type Number interface {
    ~int | ~int64 | ~float32 | ~float64
}

// Generic types
type Stack[T any] struct {
    items []T
}

// Type inference
result := Min(10, 20)  // T inferred as int

// Instantiation
var intStack Stack[int]
floatMin := Min[float64]
```

### Error Handling

```go
// Error wrapping
if err != nil {
    return fmt.Errorf("operation failed: %w", err)
}

// Error checking
if errors.Is(err, os.ErrNotExist) {
    // Handle specific error
}

// Error type assertion
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Path:", pathErr.Path)
}

// Multiple errors
errs := errors.Join(err1, err2, err3)
```

### Context Usage

```go
// Create contexts
ctx := context.Background()
ctx, cancel := context.WithCancel(ctx)
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
ctx, cancel := context.WithDeadline(ctx, deadline)
defer cancel()

// Context values
ctx = context.WithValue(ctx, key, value)
value := ctx.Value(key)

// Check cancellation
select {
case <-ctx.Done():
    return ctx.Err()
default:
    // Continue processing
}
```

### Module Commands

```bash
# Initialize module
go mod init github.com/user/project

# Add/update dependencies
go get github.com/pkg/errors@latest
go get -u ./...  # Update all

# Clean up
go mod tidy

# Vendor dependencies
go mod vendor

# Module graph
go mod graph

# Why dependency exists
go mod why github.com/pkg/errors

# Download dependencies
go mod download

# Verify dependencies
go mod verify
```

### Testing

```go
// Subtests
func TestFeature(t *testing.T) {
    t.Run("case1", func(t *testing.T) {
        // Test case 1
    })
    t.Run("case2", func(t *testing.T) {
        t.Parallel()  // Run in parallel
        // Test case 2
    })
}

// Cleanup
t.Cleanup(func() {
    // Cleanup code
})

// Fuzzing
func FuzzParse(f *testing.F) {
    f.Add("seed input")
    f.Fuzz(func(t *testing.T, input string) {
        Parse(input)
    })
}

// Skip conditions
if testing.Short() {
    t.Skip("skipping in short mode")
}

// Benchmarks
func BenchmarkFeature(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Feature()
    }
}
```

## Standard Library Updates

### log/slog (Go 1.21+)

```go
// Basic logging
slog.Info("message", "key", "value")
slog.Error("error occurred", "error", err)

// Custom logger
logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
}))

// Structured logging
logger.With("request_id", requestID).Info("processing")

// Groups
logger.LogAttrs(ctx, slog.LevelInfo, "message",
    slog.Group("user",
        slog.String("id", userID),
        slog.String("name", userName),
    ),
)
```

### slices Package (Go 1.21+)

```go
import "slices"

// Operations
slices.Sort(s)
slices.SortFunc(s, cmp)
slices.BinarySearch(s, target)
slices.Contains(s, value)
slices.Index(s, value)
slices.Equal(s1, s2)
slices.Clone(s)
slices.Compact(s)
slices.Delete(s, i, j)
slices.Insert(s, i, values...)
slices.Replace(s, i, j, values...)
slices.Reverse(s)
```

### maps Package (Go 1.21+)

```go
import "maps"

// Operations
maps.Clone(m)
maps.Copy(dst, src)
maps.Equal(m1, m2)
maps.DeleteFunc(m, fn)
keys := maps.Keys(m)
values := maps.Values(m)
```

### cmp Package (Go 1.21+)

```go
import "cmp"

// Comparisons
result := cmp.Compare(a, b)  // -1, 0, or 1
less := cmp.Less(a, b)

// Ordered constraint
func Min[T cmp.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}
```

### embed Package (Go 1.16+)

```go
import "embed"

//go:embed static/*
var staticFiles embed.FS

//go:embed template.html
var templateHTML string

//go:embed data.json
var dataJSON []byte

// Usage
data, _ := staticFiles.ReadFile("static/index.html")
http.Handle("/", http.FileServer(http.FS(staticFiles)))
```

## Build and Tool Commands

### Build Flags

```bash
# Build with optimizations
go build -ldflags="-s -w" -trimpath

# Cross-compilation
GOOS=linux GOARCH=amd64 go build
GOOS=windows GOARCH=amd64 go build
GOOS=darwin GOARCH=arm64 go build

# Build modes
go build -buildmode=plugin
go build -buildmode=c-shared
go build -buildmode=c-archive

# Build tags
go build -tags="integration"

# Race detector
go build -race

# Generate assembly
go build -gcflags="-S"
```

### Go Tool Commands

```bash
# Format code
go fmt ./...
gofmt -s -w .

# Vet code
go vet ./...

# Generate code
go generate ./...

# List packages
go list -m all
go list -json ./...

# Documentation
go doc fmt.Printf
go doc -all fmt

# Clean
go clean -cache
go clean -modcache
go clean -testcache

# Environment
go env
go env GOPATH GOROOT
```

### Testing Commands

```bash
# Run tests
go test ./...
go test -v ./...
go test -race ./...
go test -short ./...

# Coverage
go test -cover ./...
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out

# Benchmarks
go test -bench=.
go test -bench=. -benchmem
go test -bench=. -benchtime=10s
go test -bench=. -count=10

# Fuzzing
go test -fuzz=FuzzParse
go test -fuzz=FuzzParse -fuzztime=30s
```

## Common Patterns

### Options Pattern

```go
type Options struct {
    Timeout time.Duration
    Retries int
    Logger  *slog.Logger
}

type Option func(*Options)

func WithTimeout(d time.Duration) Option {
    return func(o *Options) {
        o.Timeout = d
    }
}

func NewClient(opts ...Option) *Client {
    options := &Options{
        Timeout: 30 * time.Second,
        Retries: 3,
    }
    for _, opt := range opts {
        opt(options)
    }
    return &Client{options: options}
}

// Usage
client := NewClient(
    WithTimeout(5*time.Second),
    WithRetries(5),
)
```

### Builder Pattern

```go
type ServerBuilder struct {
    host string
    port int
    tls  bool
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{
        host: "localhost",
        port: 8080,
    }
}

func (b *ServerBuilder) Host(h string) *ServerBuilder {
    b.host = h
    return b
}

func (b *ServerBuilder) Port(p int) *ServerBuilder {
    b.port = p
    return b
}

func (b *ServerBuilder) Build() *Server {
    return &Server{
        host: b.host,
        port: b.port,
        tls:  b.tls,
    }
}

// Usage
server := NewServerBuilder().
    Host("api.example.com").
    Port(443).
    Build()
```

### Worker Pool Pattern

```go
func workerPool(jobs <-chan Job, results chan<- Result, workers int) {
    var wg sync.WaitGroup
    
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }
    
    wg.Wait()
    close(results)
}
```

## HTTP Patterns

### Modern HTTP Routing (Go 1.22+)

```go
mux := http.NewServeMux()

// Method-based routing
mux.HandleFunc("GET /users", getUsers)
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("PUT /users/{id}", updateUser)
mux.HandleFunc("DELETE /users/{id}", deleteUser)

// Wildcards
mux.HandleFunc("GET /files/{path...}", serveFile)

// Path values
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // Use id
}
```

### Middleware Pattern

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, middlewares ...Middleware) http.Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func Logging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        slog.Info("request", "method", r.Method, "path", r.URL.Path, "duration", time.Since(start))
    })
}

// Usage
handler := Chain(
    mainHandler,
    Logging,
    Authentication,
    RateLimit,
)
```

## Concurrency Patterns

### sync Types

```go
// WaitGroup
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // Work
}()
wg.Wait()

// Once
var once sync.Once
once.Do(func() {
    // Initialize once
})

// Mutex
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()

// RWMutex
var rwmu sync.RWMutex
rwmu.RLock()  // For reading
defer rwmu.RUnlock()

// Map (concurrent-safe)
var m sync.Map
m.Store(key, value)
value, ok := m.Load(key)
m.Delete(key)
m.Range(func(key, value interface{}) bool {
    // Process
    return true
})

// Pool
var pool = sync.Pool{
    New: func() interface{} {
        return new(Buffer)
    },
}
buffer := pool.Get().(*Buffer)
defer pool.Put(buffer)
```

### Channel Patterns

```go
// Fan-in
func fanIn(inputs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, in := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for val := range ch {
                out <- val
            }
        }(in)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

// Fan-out
func fanOut(in <-chan int, workers int) []<-chan int {
    outs := make([]<-chan int, workers)
    
    for i := 0; i < workers; i++ {
        out := make(chan int)
        outs[i] = out
        
        go func() {
            for val := range in {
                out <- process(val)
            }
            close(out)
        }()
    }
    
    return outs
}
```

## Performance Tips

### Benchmarking

```go
// Benchmark with reset timer
func BenchmarkFeature(b *testing.B) {
    // Setup
    data := prepareData()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        Feature(data)
    }
}

// Benchmark with stop timer
func BenchmarkWithPause(b *testing.B) {
    for i := 0; i < b.N; i++ {
        b.StopTimer()
        data := expensiveSetup()
        b.StartTimer()
        
        Feature(data)
    }
}

// Parallel benchmark
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Feature()
        }
    })
}
```

### Memory Optimization

```go
// String concatenation
var sb strings.Builder
sb.WriteString("hello")
sb.WriteString(" ")
sb.WriteString("world")
result := sb.String()

// Preallocate slices
slice := make([]int, 0, expectedSize)

// Reuse buffers
var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

// Clear slice without allocation
slice = slice[:0]

// Avoid string([]byte) copies
b := []byte("text")
// Use b directly instead of string(b) when possible
```

## Security Checklist

```go
// SQL injection prevention
db.Query("SELECT * FROM users WHERE id = ?", userID)  // ✓
db.Query("SELECT * FROM users WHERE id = " + userID)  // ✗

// Command injection prevention
exec.Command("echo", userInput)  // ✓
exec.Command("sh", "-c", "echo " + userInput)  // ✗

// Path traversal prevention
path := filepath.Clean(userPath)
if !strings.HasPrefix(path, baseDir) {
    // Reject
}

// Constant time comparison
subtle.ConstantTimeCompare([]byte(a), []byte(b))

// Secure random
b := make([]byte, 32)
_, err := rand.Read(b)  // crypto/rand
```

## IDE Shortcuts (VS Code)

```
Ctrl+Space          - Trigger suggestions
F12                 - Go to definition
Shift+F12          - Find all references
Ctrl+Shift+F12     - Go to implementation
Alt+F12            - Peek definition
F2                 - Rename symbol
Ctrl+.             - Quick fix
Ctrl+Shift+P       - Command palette
Ctrl+Shift+O       - Go to symbol
Ctrl+T             - Go to symbol (workspace)
Alt+Shift+O        - Organize imports
Ctrl+K Ctrl+I      - Show hover
```

## Debugging Commands (Delve)

```bash
# Start debugging
dlv debug
dlv test
dlv attach PID

# Breakpoints
break main.go:10
break FunctionName
condition 1 x > 10

# Execution
continue (c)
next (n)
step (s)
stepout (so)
restart (r)

# Inspection
print variable
locals
args
stack
goroutines
threads

# Navigation
up
down
frame n
goroutine n
```

## Git Workflow for Go

```bash
# Initialize Go project
git init
echo "bin/" >> .gitignore
echo "*.exe" >> .gitignore
echo "vendor/" >> .gitignore
go mod init github.com/user/project
git add .
git commit -m "Initial commit"

# Feature branch workflow
git checkout -b feature/new-feature
# Make changes
go test ./...
git add .
git commit -m "Add new feature"
git push -u origin feature/new-feature

# Update dependencies
go get -u ./...
go mod tidy
git add go.mod go.sum
git commit -m "Update dependencies"
```

## Docker for Go

```dockerfile
# Multi-stage build
FROM golang:1.22-alpine AS builder
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /build/app .
CMD ["./app"]
```

## Makefile Template

```makefile
.PHONY: all build test clean run

BINARY=app
GOFILES=$(shell find . -name '*.go' -type f)

all: test build

build:
	go build -o ${BINARY} -v

test:
	go test -v -race -coverprofile=coverage.txt ./...

clean:
	go clean
	rm -f ${BINARY}
	rm -f coverage.txt

run:
	go run .

deps:
	go mod download
	go mod tidy

fmt:
	go fmt ./...
	gofumpt -l -w .

lint:
	golangci-lint run

docker:
	docker build -t ${BINARY}:latest .
```

## Environment Variables

```bash
# Go environment
GOPATH          # Go workspace (deprecated with modules)
GOROOT          # Go installation directory
GOOS            # Target OS
GOARCH          # Target architecture
CGO_ENABLED     # Enable/disable cgo
GOMAXPROCS      # Max CPUs to use
GOMEMLIMIT      # Memory limit (Go 1.19+)

# Module proxy
GOPROXY         # Module proxy URL
GONOPROXY       # Bypass proxy
GOSUMDB         # Checksum database
GONOSUMDB       # Bypass checksum
GOPRIVATE       # Private modules

# Development
GODEBUG         # Runtime debug settings
GOTRACEBACK     # Panic traceback level
```

---

*Continue to [Appendix C: Resources](appendix-c.md)*