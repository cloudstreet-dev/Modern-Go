# Appendix A: Migration Guide

## From GOPATH to Modules

The transition from GOPATH to modules represents one of the biggest changes in Go's history. This guide helps you migrate existing projects and adopt modern Go practices.

### Understanding the Transition

```bash
# Old way (pre-Go 1.11)
export GOPATH=$HOME/go
cd $GOPATH/src/github.com/user/project
go get github.com/pkg/errors

# New way (Go 1.11+)
cd ~/projects/myproject  # Anywhere outside GOPATH
go mod init github.com/user/project
go get github.com/pkg/errors
```

### Step-by-Step Migration

#### 1. Initialize Module

```bash
# Navigate to your project root
cd $GOPATH/src/github.com/user/project

# Initialize module
go mod init github.com/user/project

# This creates go.mod
module github.com/user/project

go 1.22
```

#### 2. Add Dependencies

```bash
# Download dependencies
go mod download

# Or build to automatically add dependencies
go build ./...

# Verify and clean up
go mod tidy
```

#### 3. Vendor Dependencies (Optional)

```bash
# Create vendor directory
go mod vendor

# Build using vendor
go build -mod=vendor ./...

# Verify vendor completeness
go mod verify
```

#### 4. Update CI/CD

```yaml
# Old CI configuration
before_script:
  - go get -t -v ./...

# New CI configuration  
before_script:
  - go mod download
  - go mod verify
```

### Common Migration Issues

```go
// Issue: Import path conflicts
// Solution: Use replace directive
replace github.com/old/package => github.com/new/package v1.2.3

// Issue: Private repositories
// Solution: Configure GOPRIVATE
export GOPRIVATE=github.com/company/*

// Issue: Checksum mismatches
// Solution: Clear module cache
go clean -modcache
```

## Upgrading to Generics

### Converting Interface{} to Type Parameters

```go
// Old: interface{} based code
func Max(a, b interface{}) interface{} {
    switch a := a.(type) {
    case int:
        if a > b.(int) {
            return a
        }
        return b.(int)
    case float64:
        if a > b.(float64) {
            return a
        }
        return b.(float64)
    default:
        panic("unsupported type")
    }
}

// New: Generic version
func Max[T constraints.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

### Converting Type-Specific Functions

```go
// Old: Multiple functions
func SumInts(values []int) int {
    var sum int
    for _, v := range values {
        sum += v
    }
    return sum
}

func SumFloats(values []float64) float64 {
    var sum float64
    for _, v := range values {
        sum += v
    }
    return sum
}

// New: Single generic function
func Sum[T constraints.Numeric](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}
```

### Updating Container Types

```go
// Old: Interface{} containers
type Stack struct {
    items []interface{}
}

func (s *Stack) Push(item interface{}) {
    s.items = append(s.items, item)
}

func (s *Stack) Pop() interface{} {
    if len(s.items) == 0 {
        return nil
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}

// New: Generic container
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

## Adopting New Error Handling

### Migrating to Error Wrapping

```go
// Old: Simple error returns
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err  // Lost context
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, err  // Lost context
    }
    
    return &config, nil
}

// New: Error wrapping
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading config file: %w", err)
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("parsing config JSON: %w", err)
    }
    
    return &config, nil
}
```

### Using errors.Is and errors.As

```go
// Old: Type assertions and equality checks
if err == sql.ErrNoRows {
    // Handle no rows
}

if perr, ok := err.(*os.PathError); ok {
    // Handle path error
}

// New: errors.Is and errors.As
if errors.Is(err, sql.ErrNoRows) {
    // Handle no rows
}

var perr *os.PathError
if errors.As(err, &perr) {
    // Handle path error
}
```

### Migrating to Error Groups

```go
// Old: Manual goroutine coordination
func fetchAll(urls []string) error {
    errors := make(chan error, len(urls))
    
    for _, url := range urls {
        go func(url string) {
            errors <- fetch(url)
        }(url)
    }
    
    for range urls {
        if err := <-errors; err != nil {
            return err
        }
    }
    return nil
}

// New: Using errgroup
func fetchAll(urls []string) error {
    g, ctx := errgroup.WithContext(context.Background())
    
    for _, url := range urls {
        url := url // Capture loop variable
        g.Go(func() error {
            return fetchWithContext(ctx, url)
        })
    }
    
    return g.Wait()
}
```

## Modernizing Concurrency Patterns

### Adopting Context

```go
// Old: No cancellation
func worker(jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        result := process(job)
        results <- result
    }
}

// New: Context-aware
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            return
        case job, ok := <-jobs:
            if !ok {
                return
            }
            result := process(ctx, job)
            select {
            case <-ctx.Done():
                return
            case results <- result:
            }
        }
    }
}
```

### Migrating to sync.Map

```go
// Old: Manual locking
type Cache struct {
    mu    sync.RWMutex
    items map[string]interface{}
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.items[key]
    return val, ok
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = value
}

// New: sync.Map for concurrent access
type Cache struct {
    items sync.Map
}

func (c *Cache) Get(key string) (interface{}, bool) {
    return c.items.Load(key)
}

func (c *Cache) Set(key string, value interface{}) {
    c.items.Store(key, value)
}
```

## Updating Testing Practices

### Adopting Fuzzing

```go
// Old: Property-based testing with external libraries
func TestParseManual(t *testing.T) {
    inputs := []string{
        "valid",
        "123",
        "!@#$%",
        strings.Repeat("a", 1000),
    }
    
    for _, input := range inputs {
        result, err := Parse(input)
        // Manual validation
    }
}

// New: Native fuzzing
func FuzzParse(f *testing.F) {
    // Seed corpus
    f.Add("valid")
    f.Add("123")
    f.Add("!@#$%")
    
    f.Fuzz(func(t *testing.T, input string) {
        result, err := Parse(input)
        if err != nil {
            return // Invalid input is okay
        }
        
        // Verify invariants
        if result.String() != input {
            t.Errorf("roundtrip failed: %s != %s", result.String(), input)
        }
    })
}
```

### Using T.Cleanup

```go
// Old: Manual cleanup with defer
func TestWithTempFile(t *testing.T) {
    tmpfile, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    defer os.Remove(tmpfile.Name())
    defer tmpfile.Close()
    
    // Test code
}

// New: T.Cleanup
func TestWithTempFile(t *testing.T) {
    tmpfile, err := os.CreateTemp("", "test")
    if err != nil {
        t.Fatal(err)
    }
    
    t.Cleanup(func() {
        tmpfile.Close()
        os.Remove(tmpfile.Name())
    })
    
    // Test code
}
```

## Migrating to Modern Standard Library

### Using embed.FS

```go
// Old: External files or generated code
//go:generate go-bindata -o assets.go static/...

func getAsset(name string) ([]byte, error) {
    return Asset(name)
}

// New: embed package
//go:embed static/*
var staticFiles embed.FS

func getAsset(name string) ([]byte, error) {
    return staticFiles.ReadFile(name)
}
```

### Adopting log/slog

```go
// Old: log package or third-party loggers
log.Printf("processing user id=%d name=%s", user.ID, user.Name)

// Or with logrus
logrus.WithFields(logrus.Fields{
    "id":   user.ID,
    "name": user.Name,
}).Info("processing user")

// New: slog
slog.Info("processing user",
    "id", user.ID,
    "name", user.Name,
)
```

### Using New Slice/Map Functions

```go
// Old: Manual operations
// Clone slice
copied := make([]int, len(original))
copy(copied, original)

// Check if contains
found := false
for _, v := range slice {
    if v == target {
        found = true
        break
    }
}

// New: slices package
copied := slices.Clone(original)
found := slices.Contains(slice, target)

// Old: Manual map operations
maps2 := make(map[string]int)
for k, v := range maps1 {
    maps2[k] = v
}

// New: maps package
maps2 := maps.Clone(maps1)
```

## Performance Migration Checklist

### 1. Profile Before Migrating
```bash
# Baseline performance
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof
```

### 2. Gradual Migration
```go
// Use build tags for gradual rollout
//go:build newfeature

// Keep old implementation available
//go:build !newfeature
```

### 3. Benchmark Comparisons
```bash
# Compare old vs new
benchstat old.txt new.txt
```

### 4. Memory Considerations
```go
// Check for allocation changes
go test -bench=. -benchmem
```

## Tool Migration

### Updating Build Scripts

```bash
# Old Makefile
build:
    go build -v

# New Makefile with modern flags
build:
    go build -v -trimpath -ldflags="-s -w" -o bin/app

# Old Docker
FROM golang:1.10
WORKDIR /go/src/app
COPY . .
RUN go get -d -v ./...
RUN go install -v ./...

# New Docker with modules
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o app

FROM gcr.io/distroless/base
COPY --from=builder /app/app /
CMD ["/app"]
```

### CI/CD Updates

```yaml
# Old GitHub Actions
- uses: actions/setup-go@v2
  with:
    go-version: 1.10
- run: go get -v -t -d ./...
- run: go test -v ./...

# New GitHub Actions
- uses: actions/setup-go@v4
  with:
    go-version: '1.22'
    cache: true
- run: go mod download
- run: go test -v -race ./...
```

## Migration Timeline

### Phase 1: Foundation (Weeks 1-2)
- Update to latest Go version
- Migrate from GOPATH to modules
- Update CI/CD pipelines

### Phase 2: Core Updates (Weeks 3-4)
- Adopt error wrapping
- Add context to APIs
- Update logging to slog

### Phase 3: Optimizations (Weeks 5-6)
- Convert to generics where beneficial
- Use new standard library packages
- Implement fuzzing tests

### Phase 4: Polish (Weeks 7-8)
- Performance tuning
- Documentation updates
- Team training

## Common Pitfalls

### 1. Loop Variable Capture
```go
// Problem in older Go versions
for _, v := range values {
    go func() {
        process(v) // Bug: captures loop variable
    }()
}

// Solution (Go 1.22+ fixes this)
for _, v := range values {
    v := v // Capture
    go func() {
        process(v)
    }()
}
```

### 2. Module Version Conflicts
```go
// Problem: Incompatible versions
// Solution: Use MVS (Minimal Version Selection)
go get -u ./...  // Update all dependencies
go mod tidy      // Clean up
```

### 3. Breaking Changes
```go
// Always check release notes
// Use go fix for automatic updates
go fix ./...
```

## Resources for Migration

- [Go Modules Documentation](https://go.dev/doc/modules/)
- [Go 1.18 Release Notes](https://go.dev/doc/go1.18) (Generics)
- [Error Handling Best Practices](https://go.dev/blog/go1.13-errors)
- [Migration Tools](https://github.com/golang/tools)
- [Community Migration Guides](https://github.com/golang/go/wiki/Modules)

## Summary

Migration to modern Go is a journey, not a destination. Start with the fundamentals (modules, error handling), then gradually adopt new features (generics, fuzzing) as they make sense for your project. The Go community values backward compatibility, so you can migrate at your own pace while maintaining a working codebase.

---

*Continue to [Appendix B: Quick Reference](appendix-b.md)*