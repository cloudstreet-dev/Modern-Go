# Chapter 10: Advanced Testing

## Testing Grows Up

Go's testing philosophy started simple: `go test` and `*_test.go` files. But modern Go testing has evolved into a sophisticated ecosystem with fuzzing, improved benchmarks, test helpers, and better coverage tools. Testing isn't just about finding bugs anymore—it's about proving correctness, measuring performance, and documenting behavior.

## Native Fuzzing (Go 1.18+)

Fuzzing automatically generates test inputs to find edge cases:

```go
// Basic fuzzing test
func FuzzParseQuery(f *testing.F) {
    // Seed corpus - initial test cases
    f.Add("user=alice&age=30")
    f.Add("name=bob&city=NYC")
    f.Add("")
    f.Add("invalid&&&&")
    
    // The fuzz target
    f.Fuzz(func(t *testing.T, input string) {
        result, err := ParseQuery(input)
        
        if err != nil {
            // Error is acceptable for malformed input
            return
        }
        
        // Verify properties that should always hold
        encoded := result.Encode()
        reparsed, err := ParseQuery(encoded)
        if err != nil {
            t.Fatalf("Failed to reparse encoded query: %v", err)
        }
        
        if !reflect.DeepEqual(result, reparsed) {
            t.Errorf("Roundtrip failed: %v != %v", result, reparsed)
        }
    })
}

// Run fuzzing
// $ go test -fuzz=FuzzParseQuery
// $ go test -fuzz=FuzzParseQuery -fuzztime=10s

// Advanced fuzzing with multiple inputs
func FuzzJSONTransform(f *testing.F) {
    // Seed with various JSON structures
    f.Add(`{"name":"Alice","age":30}`, "name")
    f.Add(`{"items":[1,2,3]}`, "items.0")
    f.Add(`{"nested":{"deep":{"value":42}}}`, "nested.deep.value")
    
    f.Fuzz(func(t *testing.T, jsonStr string, path string) {
        var data interface{}
        if err := json.Unmarshal([]byte(jsonStr), &data); err != nil {
            t.Skip()  // Skip invalid JSON
        }
        
        // Test the function
        result, err := GetJSONPath(data, path)
        
        // Property: valid path should not panic
        if err != nil && strings.Contains(err.Error(), "panic") {
            t.Fatal("Function panicked")
        }
        
        // Property: result should be serializable
        if result != nil {
            if _, err := json.Marshal(result); err != nil {
                t.Errorf("Result not serializable: %v", err)
            }
        }
    })
}

// Fuzzing with custom types
func FuzzBinaryProtocol(f *testing.F) {
    // Add binary seed data
    f.Add([]byte{0x01, 0x02, 0x03})
    f.Add([]byte{0xFF, 0xFE, 0xFD})
    
    f.Fuzz(func(t *testing.T, data []byte) {
        // Try to decode
        msg, err := DecodeMessage(data)
        if err != nil {
            return  // Invalid message is OK
        }
        
        // Re-encode
        encoded := msg.Encode()
        
        // Decode again
        msg2, err := DecodeMessage(encoded)
        if err != nil {
            t.Fatalf("Failed to decode re-encoded message: %v", err)
        }
        
        // Should match
        if !msg.Equal(msg2) {
            t.Error("Message changed after encode/decode cycle")
        }
    })
}
```

## Test Helpers and Cleanup

Modern test helpers for cleaner tests:

```go
// Test helpers with t.Helper()
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()  // Marks this as a helper function
    if !reflect.DeepEqual(got, want) {
        t.Errorf("got %v, want %v", got, want)
    }
}

func createTestServer(t *testing.T) *Server {
    t.Helper()
    
    // Setup
    server := &Server{
        Port: getFreePort(t),
    }
    
    // Register cleanup
    t.Cleanup(func() {
        server.Shutdown()
    })
    
    if err := server.Start(); err != nil {
        t.Fatalf("Failed to start server: %v", err)
    }
    
    return server
}

// Using helpers in tests
func TestServerResponse(t *testing.T) {
    server := createTestServer(t)  // Cleanup automatic
    
    resp, err := http.Get(server.URL + "/health")
    requireNoError(t, err)
    assertEqual(t, resp.StatusCode, 200)
}

// Test fixtures with cleanup
func setupDatabase(t *testing.T) *sql.DB {
    t.Helper()
    
    // Create temp database
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("Failed to create test database: %v", err)
    }
    
    // Run migrations
    if err := runMigrations(db); err != nil {
        t.Fatalf("Failed to run migrations: %v", err)
    }
    
    // Cleanup
    t.Cleanup(func() {
        db.Close()
    })
    
    return db
}

// Temporary directories with automatic cleanup
func TestFileProcessing(t *testing.T) {
    // t.TempDir() creates a temp directory and removes it after test
    tmpDir := t.TempDir()
    
    // Create test files
    testFile := filepath.Join(tmpDir, "test.txt")
    err := os.WriteFile(testFile, []byte("test content"), 0644)
    requireNoError(t, err)
    
    // Test file processing
    result := ProcessFile(testFile)
    assertEqual(t, result, "processed: test content")
}
```

## Table-Driven Tests Enhanced

Modern patterns for table-driven tests:

```go
func TestCalculator(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        op       string
        want     int
        wantErr  bool
        setup    func(*testing.T)
        teardown func(*testing.T)
    }{
        {
            name: "addition",
            a:    5,
            b:    3,
            op:   "+",
            want: 8,
        },
        {
            name: "division",
            a:    10,
            b:    2,
            op:   "/",
            want: 5,
        },
        {
            name:    "division by zero",
            a:       10,
            b:       0,
            op:      "/",
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Parallel execution for independent tests
            t.Parallel()
            
            if tt.setup != nil {
                tt.setup(t)
            }
            
            if tt.teardown != nil {
                t.Cleanup(tt.teardown)
            }
            
            got, err := Calculate(tt.a, tt.b, tt.op)
            
            if tt.wantErr {
                if err == nil {
                    t.Errorf("expected error, got none")
                }
                return
            }
            
            if err != nil {
                t.Errorf("unexpected error: %v", err)
                return
            }
            
            if got != tt.want {
                t.Errorf("got %d, want %d", got, tt.want)
            }
        })
    }
}

// Generic test table
type TestCase[Input, Output any] struct {
    Name     string
    Input    Input
    Expected Output
    Error    error
}

func RunTests[I, O any](t *testing.T, fn func(I) (O, error), cases []TestCase[I, O]) {
    for _, tc := range cases {
        t.Run(tc.Name, func(t *testing.T) {
            got, err := fn(tc.Input)
            
            if tc.Error != nil {
                if err == nil || err.Error() != tc.Error.Error() {
                    t.Errorf("expected error %v, got %v", tc.Error, err)
                }
                return
            }
            
            if err != nil {
                t.Errorf("unexpected error: %v", err)
                return
            }
            
            if !reflect.DeepEqual(got, tc.Expected) {
                t.Errorf("got %v, want %v", got, tc.Expected)
            }
        })
    }
}
```

## Benchmark Improvements

Advanced benchmarking techniques:

```go
// Benchmark with size variations
func BenchmarkSort(b *testing.B) {
    sizes := []int{10, 100, 1000, 10000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size-%d", size), func(b *testing.B) {
            // Create data outside the benchmark loop
            data := make([]int, size)
            for i := range data {
                data[i] = rand.Intn(size)
            }
            
            b.ResetTimer()  // Don't count setup time
            b.ReportAllocs()  // Report memory allocations
            
            for i := 0; i < b.N; i++ {
                // Copy data for each iteration
                tmp := make([]int, len(data))
                copy(tmp, data)
                
                sort.Ints(tmp)
            }
            
            // Report custom metrics
            b.ReportMetric(float64(size), "items")
            b.ReportMetric(float64(b.Elapsed().Nanoseconds())/float64(b.N*size), "ns/item")
        })
    }
}

// Parallel benchmarks
func BenchmarkConcurrentMap(b *testing.B) {
    m := NewConcurrentMap()
    
    b.RunParallel(func(pb *testing.PB) {
        // Each goroutine gets its own counter
        i := 0
        for pb.Next() {
            key := fmt.Sprintf("key-%d", i)
            m.Set(key, i)
            m.Get(key)
            i++
        }
    })
}

// Benchmark comparisons
func BenchmarkMaps(b *testing.B) {
    b.Run("sync.Map", func(b *testing.B) {
        var m sync.Map
        b.RunParallel(func(pb *testing.PB) {
            i := 0
            for pb.Next() {
                m.Store(i, i)
                m.Load(i)
                i++
            }
        })
    })
    
    b.Run("RWMutex+map", func(b *testing.B) {
        m := make(map[int]int)
        var mu sync.RWMutex
        
        b.RunParallel(func(pb *testing.PB) {
            i := 0
            for pb.Next() {
                mu.Lock()
                m[i] = i
                mu.Unlock()
                
                mu.RLock()
                _ = m[i]
                mu.RUnlock()
                i++
            }
        })
    })
}

// Memory-focused benchmarks
func BenchmarkMemory(b *testing.B) {
    b.Run("with-pool", func(b *testing.B) {
        pool := sync.Pool{
            New: func() interface{} {
                return make([]byte, 1024)
            },
        }
        
        b.ReportAllocs()
        b.ResetTimer()
        
        for i := 0; i < b.N; i++ {
            buf := pool.Get().([]byte)
            // Use buffer
            _ = buf[:10]
            pool.Put(buf)
        }
    })
    
    b.Run("without-pool", func(b *testing.B) {
        b.ReportAllocs()
        b.ResetTimer()
        
        for i := 0; i < b.N; i++ {
            buf := make([]byte, 1024)
            // Use buffer
            _ = buf[:10]
        }
    })
}
```

## Testing HTTP Services

Modern HTTP testing patterns:

```go
// HTTP handler testing
func TestHandler(t *testing.T) {
    handler := http.HandlerFunc(MyHandler)
    
    tests := []struct {
        name       string
        method     string
        path       string
        body       string
        wantStatus int
        wantBody   string
        wantHeader map[string]string
    }{
        {
            name:       "GET success",
            method:     "GET",
            path:       "/api/users/123",
            wantStatus: http.StatusOK,
            wantBody:   `{"id":"123","name":"Alice"}`,
            wantHeader: map[string]string{
                "Content-Type": "application/json",
            },
        },
        {
            name:       "POST with body",
            method:     "POST",
            path:       "/api/users",
            body:       `{"name":"Bob"}`,
            wantStatus: http.StatusCreated,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest(tt.method, tt.path, 
                strings.NewReader(tt.body))
            rec := httptest.NewRecorder()
            
            handler.ServeHTTP(rec, req)
            
            if rec.Code != tt.wantStatus {
                t.Errorf("status = %d, want %d", rec.Code, tt.wantStatus)
            }
            
            if tt.wantBody != "" {
                got := strings.TrimSpace(rec.Body.String())
                if got != tt.wantBody {
                    t.Errorf("body = %s, want %s", got, tt.wantBody)
                }
            }
            
            for key, want := range tt.wantHeader {
                if got := rec.Header().Get(key); got != want {
                    t.Errorf("header[%s] = %s, want %s", key, got, want)
                }
            }
        })
    }
}

// Integration testing with real server
func TestIntegration(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    // Start test server
    srv := httptest.NewServer(http.HandlerFunc(MyHandler))
    defer srv.Close()
    
    client := srv.Client()
    
    // Test real HTTP calls
    resp, err := client.Get(srv.URL + "/api/health")
    if err != nil {
        t.Fatal(err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        t.Errorf("status = %d, want 200", resp.StatusCode)
    }
}

// Testing with contexts and timeouts
func TestWithTimeout(t *testing.T) {
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()
    
    done := make(chan bool)
    
    go func() {
        // Simulate slow operation
        time.Sleep(200 * time.Millisecond)
        done <- true
    }()
    
    select {
    case <-done:
        t.Fatal("operation should have timed out")
    case <-ctx.Done():
        // Expected timeout
    }
}
```

## Mocking and Interfaces

Modern mocking patterns:

```go
// Interface for mocking
type DataStore interface {
    Get(ctx context.Context, key string) (string, error)
    Set(ctx context.Context, key, value string) error
}

// Mock implementation
type MockDataStore struct {
    GetFunc func(ctx context.Context, key string) (string, error)
    SetFunc func(ctx context.Context, key, value string) error
    
    // Track calls for assertions
    GetCalls []string
    SetCalls []struct{ Key, Value string }
    mu       sync.Mutex
}

func (m *MockDataStore) Get(ctx context.Context, key string) (string, error) {
    m.mu.Lock()
    m.GetCalls = append(m.GetCalls, key)
    m.mu.Unlock()
    
    if m.GetFunc != nil {
        return m.GetFunc(ctx, key)
    }
    return "", nil
}

func (m *MockDataStore) Set(ctx context.Context, key, value string) error {
    m.mu.Lock()
    m.SetCalls = append(m.SetCalls, struct{ Key, Value string }{key, value})
    m.mu.Unlock()
    
    if m.SetFunc != nil {
        return m.SetFunc(ctx, key, value)
    }
    return nil
}

// Using the mock
func TestService(t *testing.T) {
    mock := &MockDataStore{
        GetFunc: func(ctx context.Context, key string) (string, error) {
            if key == "exists" {
                return "value", nil
            }
            return "", errors.New("not found")
        },
    }
    
    service := NewService(mock)
    
    result, err := service.Process(context.Background(), "exists")
    if err != nil {
        t.Fatal(err)
    }
    
    // Assert calls were made
    if len(mock.GetCalls) != 1 {
        t.Errorf("expected 1 Get call, got %d", len(mock.GetCalls))
    }
    
    if mock.GetCalls[0] != "exists" {
        t.Errorf("Get called with %s, want 'exists'", mock.GetCalls[0])
    }
}

// Generic mock builder
type Mock[T any] struct {
    calls []CallInfo
    mu    sync.Mutex
}

type CallInfo struct {
    Method string
    Args   []interface{}
    Result []interface{}
}

func (m *Mock[T]) RecordCall(method string, args []interface{}, results ...interface{}) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    m.calls = append(m.calls, CallInfo{
        Method: method,
        Args:   args,
        Result: results,
    })
}

func (m *Mock[T]) AssertCalled(t *testing.T, method string, times int) {
    m.mu.Lock()
    defer m.mu.Unlock()
    
    count := 0
    for _, call := range m.calls {
        if call.Method == method {
            count++
        }
    }
    
    if count != times {
        t.Errorf("method %s called %d times, want %d", method, count, times)
    }
}
```

## Test Coverage Analysis

Advanced coverage techniques:

```go
// Run with coverage
// $ go test -cover
// $ go test -coverprofile=coverage.out
// $ go tool cover -html=coverage.out

// Coverage-guided testing
func TestFullCoverage(t *testing.T) {
    tests := []struct {
        name      string
        input     interface{}
        wantPanic bool
    }{
        {"nil input", nil, true},
        {"empty string", "", false},
        {"valid input", "test", false},
        {"special chars", "!@#$", false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if tt.wantPanic {
                defer func() {
                    if r := recover(); r == nil {
                        t.Error("expected panic")
                    }
                }()
            }
            
            result := ProcessInput(tt.input)
            
            if !tt.wantPanic && result == "" {
                t.Error("unexpected empty result")
            }
        })
    }
}

// Build tags for test coverage
//go:build !nocoverage
// +build !nocoverage

func TestExpensiveOperation(t *testing.T) {
    // Only run in coverage mode
    if os.Getenv("COVERAGE") != "1" {
        t.Skip("skipping expensive test")
    }
    
    // Expensive test...
}
```

## Testing Patterns

### Golden Files

Test against expected output files:

```go
var update = flag.Bool("update", false, "update golden files")

func TestGolden(t *testing.T) {
    input := loadTestData(t, "input.json")
    actual := Transform(input)
    
    golden := filepath.Join("testdata", "output.golden")
    
    if *update {
        err := os.WriteFile(golden, actual, 0644)
        if err != nil {
            t.Fatal(err)
        }
        return
    }
    
    expected, err := os.ReadFile(golden)
    if err != nil {
        t.Fatal(err)
    }
    
    if !bytes.Equal(actual, expected) {
        t.Errorf("output doesn't match golden file")
        
        // Write actual for debugging
        actualFile := golden + ".actual"
        os.WriteFile(actualFile, actual, 0644)
        t.Logf("Actual output written to %s", actualFile)
    }
}
```

### Snapshot Testing

```go
type Snapshot struct {
    t        *testing.T
    dir      string
    updateMode bool
}

func NewSnapshot(t *testing.T) *Snapshot {
    return &Snapshot{
        t:          t,
        dir:        filepath.Join("testdata", "snapshots"),
        updateMode: os.Getenv("UPDATE_SNAPSHOTS") == "1",
    }
}

func (s *Snapshot) Match(name string, data interface{}) {
    s.t.Helper()
    
    // Serialize data
    serialized, err := json.MarshalIndent(data, "", "  ")
    if err != nil {
        s.t.Fatal(err)
    }
    
    filename := filepath.Join(s.dir, name+".json")
    
    if s.updateMode {
        os.MkdirAll(s.dir, 0755)
        err := os.WriteFile(filename, serialized, 0644)
        if err != nil {
            s.t.Fatal(err)
        }
        s.t.Logf("Updated snapshot: %s", filename)
        return
    }
    
    expected, err := os.ReadFile(filename)
    if err != nil {
        s.t.Fatalf("Snapshot not found: %s (run with UPDATE_SNAPSHOTS=1)", filename)
    }
    
    if !bytes.Equal(serialized, expected) {
        s.t.Errorf("Snapshot mismatch for %s", name)
    }
}
```

### Property-Based Testing

```go
func TestProperties(t *testing.T) {
    // Property: reverse(reverse(x)) == x
    propertyReverseIdempotent := func(s string) bool {
        reversed := Reverse(s)
        doubleReversed := Reverse(reversed)
        return s == doubleReversed
    }
    
    // Property: len(concat(a, b)) == len(a) + len(b)
    propertyConcatLength := func(a, b string) bool {
        result := Concat(a, b)
        return len(result) == len(a)+len(b)
    }
    
    // Test with random inputs
    for i := 0; i < 100; i++ {
        s := randomString(rand.Intn(100))
        if !propertyReverseIdempotent(s) {
            t.Errorf("Property failed for: %s", s)
        }
        
        a := randomString(rand.Intn(50))
        b := randomString(rand.Intn(50))
        if !propertyConcatLength(a, b) {
            t.Errorf("Property failed for: %s, %s", a, b)
        }
    }
}
```

## Test Organization

### Test Suites

```go
type Suite struct {
    t  *testing.T
    db *sql.DB
}

func NewSuite(t *testing.T) *Suite {
    db := setupDatabase(t)
    return &Suite{t: t, db: db}
}

func (s *Suite) TestUser() {
    s.t.Run("Create", s.testCreateUser)
    s.t.Run("Update", s.testUpdateUser)
    s.t.Run("Delete", s.testDeleteUser)
}

func (s *Suite) testCreateUser(t *testing.T) {
    user := &User{Name: "Alice"}
    err := CreateUser(s.db, user)
    if err != nil {
        t.Fatal(err)
    }
    
    if user.ID == 0 {
        t.Error("expected user ID to be set")
    }
}

func TestSuite(t *testing.T) {
    suite := NewSuite(t)
    suite.TestUser()
}
```

## Best Practices

### 1. Fast Tests First
```go
func TestFast(t *testing.T) {
    // Quick unit tests
}

func TestSlow(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping slow test")
    }
    // Integration tests
}
```

### 2. Descriptive Names
```go
func TestUserService_CreateUser_WithValidInput_ReturnsUserWithID(t *testing.T)
```

### 3. Isolate Tests
```go
func TestIsolated(t *testing.T) {
    t.Parallel()  // Run in parallel
    // Each test gets its own resources
}
```

### 4. Use testdata Directory
```
project/
├── main.go
├── main_test.go
└── testdata/
    ├── input.json
    └── expected.json
```

## Exercises

1. **Fuzz Testing**: Write a fuzzer for a JSON parser that finds malformed inputs.

2. **Benchmark Suite**: Create comprehensive benchmarks comparing different data structures.

3. **Mock Generator**: Build a code generator that creates mocks from interfaces.

4. **Test Framework**: Design a BDD-style testing framework using Go's testing package.

5. **Coverage Reporter**: Create a tool that generates detailed coverage reports with suggestions.

## Summary

Modern Go testing has evolved from simple unit tests to a comprehensive testing ecosystem. Fuzzing finds bugs you didn't know existed, benchmarks measure real performance, and helpers make tests cleaner and more maintainable. The key is using the right tool for each testing need.

Key takeaways:
- Fuzzing automatically finds edge cases
- Test helpers reduce boilerplate
- Benchmarks should measure what matters
- Table-driven tests scale well
- Coverage is a tool, not a goal

Next, we'll explore static analysis and linting tools that catch bugs before they reach testing.

---

*Continue to [Chapter 11: Static Analysis and Linting](chapter11.md)*