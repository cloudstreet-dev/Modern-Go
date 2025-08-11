# Chapter 13: Enhanced Existing Packages

## Evolution, Not Revolution

While new packages grab headlines, the quiet improvements to existing packages often have more impact. Every Go release enhances the standard library with new methods, better performance, and refined APIs. This chapter explores how familiar packages have grown more powerful since 2015.

## net/http: The Web Framework That Isn't

The net/http package has gained so many features it rivals dedicated web frameworks:

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "time"
)

// Modern routing with patterns (Go 1.22+)
func modernRouting() {
    mux := http.NewServeMux()
    
    // Method-specific routes
    mux.HandleFunc("GET /users", listUsers)
    mux.HandleFunc("POST /users", createUser)
    mux.HandleFunc("GET /users/{id}", getUser)
    mux.HandleFunc("PUT /users/{id}", updateUser)
    mux.HandleFunc("DELETE /users/{id}", deleteUser)
    
    // Wildcards and parameters
    mux.HandleFunc("GET /files/{path...}", serveFile)
    mux.HandleFunc("GET /api/v{version}/status", apiStatus)
    
    // Host-specific routing
    mux.HandleFunc("GET admin.example.com/", adminPanel)
    mux.HandleFunc("GET {subdomain}.example.com/", handleSubdomain)
    
    // Priority routing (more specific wins)
    mux.HandleFunc("GET /static/", staticHandler)
    mux.HandleFunc("GET /static/special.js", specialHandler)  // This wins for special.js
    
    http.ListenAndServe(":8080", mux)
}

// Extract path parameters
func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Extract {id} parameter
    
    user, err := fetchUser(id)
    if err != nil {
        http.Error(w, "User not found", http.StatusNotFound)
        return
    }
    
    // Modern response helpers
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(user)
}

// Wildcard paths
func serveFile(w http.ResponseWriter, r *http.Request) {
    path := r.PathValue("path")  // Gets everything after /files/
    
    // Serve file from embedded filesystem
    http.ServeFileFS(w, r, embedFS, path)
}

// HTTP/2 and HTTP/3 support
func modernProtocols() {
    // HTTP/2 is automatic with TLS
    server := &http.Server{
        Addr: ":443",
        Handler: http.HandlerFunc(handler),
        
        // HTTP/2 configuration
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
        
        // Enable HTTP/2 push
        TLSNextProto: make(map[string]func(*http.Server, *tls.Conn, http.Handler)),
    }
    
    // HTTP/3 support (experimental)
    server.ListenAndServeTLS("cert.pem", "key.pem")
}

// Context integration improvements
func contextIntegration(w http.ResponseWriter, r *http.Request) {
    // Request context with values
    ctx := r.Context()
    
    // Add values to context
    ctx = context.WithValue(ctx, "request_id", generateID())
    r = r.WithContext(ctx)
    
    // NewRequestWithContext for client requests
    req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com", nil)
    
    // Context cancellation
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    // Make request with context
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusGatewayTimeout)
        }
        return
    }
    defer resp.Body.Close()
}

// MaxBytes reader for request size limits
func requestLimits(w http.ResponseWriter, r *http.Request) {
    // Limit request body size
    r.Body = http.MaxBytesReader(w, r.Body, 1<<20)  // 1MB limit
    
    decoder := json.NewDecoder(r.Body)
    decoder.DisallowUnknownFields()  // Strict JSON parsing
    
    var data RequestData
    if err := decoder.Decode(&data); err != nil {
        if err.Error() == "http: request body too large" {
            http.Error(w, "Request too large", http.StatusRequestEntityTooLarge)
        } else {
            http.Error(w, "Invalid JSON", http.StatusBadRequest)
        }
        return
    }
}

// ResponseController for advanced response handling (Go 1.20+)
func responseController(w http.ResponseWriter, r *http.Request) {
    rc := http.NewResponseController(w)
    
    // Set write deadline
    if err := rc.SetWriteDeadline(time.Now().Add(10 * time.Second)); err != nil {
        log.Printf("Failed to set deadline: %v", err)
    }
    
    // Flush response
    if err := rc.Flush(); err != nil {
        log.Printf("Failed to flush: %v", err)
    }
    
    // Hijack connection for WebSocket
    if hijacker, ok := w.(http.Hijacker); ok {
        conn, bufrw, err := hijacker.Hijack()
        if err != nil {
            http.Error(w, "Hijacking not supported", http.StatusInternalServerError)
            return
        }
        defer conn.Close()
        
        // Use raw connection for WebSocket or other protocols
        handleWebSocket(conn, bufrw)
    }
}

// Client improvements
func modernClient() {
    // Connection pooling and HTTP/2
    client := &http.Client{
        Transport: &http.Transport{
            MaxIdleConns:        100,
            MaxIdleConnsPerHost: 10,
            MaxConnsPerHost:     10,
            IdleConnTimeout:     90 * time.Second,
            
            // HTTP/2 configuration
            ForceAttemptHTTP2: true,
            
            // Connection reuse
            DisableKeepAlives: false,
            
            // Proxy configuration
            Proxy: http.ProxyFromEnvironment,
            
            // TLS configuration
            TLSClientConfig: &tls.Config{
                MinVersion: tls.VersionTLS12,
            },
        },
        
        // Client timeout
        Timeout: 30 * time.Second,
        
        // Cookie jar for session management
        Jar: http.DefaultCookieJar,
    }
    
    // Clone request for retry
    req, _ := http.NewRequest("GET", "https://api.example.com", nil)
    req2 := req.Clone(context.Background())
    
    // Do with context
    resp, err := client.Do(req)
}
```

## encoding/json: More Flexible, More Powerful

JSON handling has gained numerous improvements:

```go
package main

import (
    "encoding/json"
    "fmt"
    "strings"
)

// Number type for precise numeric handling
func jsonNumbers() {
    data := `{"id": 12345678901234567890, "price": 99.99}`
    
    var result map[string]json.Number
    json.Unmarshal([]byte(data), &result)
    
    // Preserve precision
    id, _ := result["id"].Int64()
    price, _ := result["price"].Float64()
    
    // Or keep as string for exact representation
    idStr := result["id"].String()
}

// DisallowUnknownFields for strict parsing
func strictJSON() {
    data := `{"name": "Alice", "age": 30, "unknown": "field"}`
    
    var person struct {
        Name string `json:"name"`
        Age  int    `json:"age"`
    }
    
    decoder := json.NewDecoder(strings.NewReader(data))
    decoder.DisallowUnknownFields()
    
    if err := decoder.Decode(&person); err != nil {
        // Error: unknown field "unknown"
        fmt.Println("Strict parsing error:", err)
    }
}

// RawMessage for delayed parsing
func rawMessage() {
    type Event struct {
        Type    string          `json:"type"`
        Payload json.RawMessage `json:"payload"`
    }
    
    data := `{
        "type": "user_created",
        "payload": {"id": 123, "name": "Bob"}
    }`
    
    var event Event
    json.Unmarshal([]byte(data), &event)
    
    // Parse payload based on type
    switch event.Type {
    case "user_created":
        var user User
        json.Unmarshal(event.Payload, &user)
    case "order_placed":
        var order Order
        json.Unmarshal(event.Payload, &order)
    }
}

// Custom marshaling with performance
type OptimizedUser struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email,omitempty"`
    IsActive bool   `json:"is_active"`
    
    // Precomputed JSON
    jsonCache []byte
}

func (u *OptimizedUser) MarshalJSON() ([]byte, error) {
    if u.jsonCache != nil {
        return u.jsonCache, nil
    }
    
    // Use alias to avoid recursion
    type Alias OptimizedUser
    data, err := json.Marshal((*Alias)(u))
    if err != nil {
        return nil, err
    }
    
    u.jsonCache = data
    return data, nil
}

// Streaming JSON arrays
func streamJSON(w io.Writer, items <-chan Item) error {
    encoder := json.NewEncoder(w)
    
    // Start array
    if _, err := w.Write([]byte("[")); err != nil {
        return err
    }
    
    first := true
    for item := range items {
        if !first {
            if _, err := w.Write([]byte(",")); err != nil {
                return err
            }
        }
        
        if err := encoder.Encode(item); err != nil {
            return err
        }
        
        first = false
    }
    
    // End array
    _, err := w.Write([]byte("]"))
    return err
}

// Valid JSON checking without parsing
func validateJSON(data []byte) bool {
    return json.Valid(data)
}

// Compact and Indent improvements
func formatJSON() {
    data := []byte(`{
        "name": "Alice",
        "age": 30
    }`)
    
    // Compact (remove whitespace)
    var compact bytes.Buffer
    json.Compact(&compact, data)
    
    // Indent with custom prefix
    var indented bytes.Buffer
    json.Indent(&indented, compact.Bytes(), "  ", "    ")
}
```

## time Package: Time Travel Made Easy

Time handling has become more sophisticated:

```go
package main

import (
    "fmt"
    "time"
)

// Monotonic time for accurate measurements
func monotonicTime() {
    start := time.Now()  // Contains both wall and monotonic time
    
    // Do work...
    time.Sleep(100 * time.Millisecond)
    
    elapsed := time.Since(start)  // Uses monotonic time, immune to clock adjustments
    fmt.Printf("Elapsed: %v\n", elapsed)
    
    // Compare times
    t1 := time.Now()
    t2 := time.Now()
    
    if t2.After(t1) {  // Comparison uses monotonic time
        fmt.Println("t2 is after t1")
    }
}

// Layout constants additions
func timeLayouts() {
    now := time.Now()
    
    // New layout constants
    fmt.Println(now.Format(time.DateTime))     // "2006-01-02 15:04:05"
    fmt.Println(now.Format(time.DateOnly))     // "2006-01-02"
    fmt.Println(now.Format(time.TimeOnly))     // "15:04:05"
    fmt.Println(now.Format(time.Kitchen))      // "3:04PM"
    fmt.Println(now.Format(time.Stamp))        // "Jan _2 15:04:05"
    fmt.Println(now.Format(time.StampMilli))   // "Jan _2 15:04:05.000"
    fmt.Println(now.Format(time.StampMicro))   // "Jan _2 15:04:05.000000"
    fmt.Println(now.Format(time.StampNano))    // "Jan _2 15:04:05.000000000"
}

// UnixMilli and UnixMicro (Go 1.17+)
func unixTimeExtensions() {
    now := time.Now()
    
    // Milliseconds
    millis := now.UnixMilli()
    fromMillis := time.UnixMilli(millis)
    
    // Microseconds
    micros := now.UnixMicro()
    fromMicros := time.UnixMicro(micros)
    
    // Compare with traditional
    seconds := now.Unix()
    nanos := now.UnixNano()
}

// Time arithmetic improvements
func timeArithmetic() {
    now := time.Now()
    
    // Add with multiple units
    future := now.Add(24 * time.Hour).Add(30 * time.Minute)
    
    // Truncate and Round
    truncated := now.Truncate(time.Hour)    // Start of hour
    rounded := now.Round(15 * time.Minute)  // Round to nearest 15 min
    
    // Check if zero
    var t time.Time
    if t.IsZero() {
        fmt.Println("Time is zero value")
    }
    
    // Compare with Equal (handles different locations)
    t1 := time.Now()
    t2 := t1.In(time.UTC)
    if t1.Equal(t2) {  // True even though different zones
        fmt.Println("Times are equal")
    }
}

// Timer and Ticker improvements
func timerImprovements() {
    // Timer with Stop and Reset
    timer := time.NewTimer(5 * time.Second)
    
    if !timer.Stop() {
        // Drain channel if timer already fired
        select {
        case <-timer.C:
        default:
        }
    }
    
    // Reset for reuse
    timer.Reset(10 * time.Second)
    
    // AfterFunc for callbacks
    afterFunc := time.AfterFunc(2*time.Second, func() {
        fmt.Println("Timer fired!")
    })
    
    // Can stop AfterFunc
    afterFunc.Stop()
    
    // Ticker with Reset (Go 1.15+)
    ticker := time.NewTicker(time.Second)
    ticker.Reset(500 * time.Millisecond)  // Change interval
    
    defer ticker.Stop()
}

// Context integration
func timeWithContext(ctx context.Context) {
    // Create timer that respects context
    timer := time.NewTimer(5 * time.Second)
    defer timer.Stop()
    
    select {
    case <-ctx.Done():
        fmt.Println("Context cancelled")
    case <-timer.C:
        fmt.Println("Timer expired")
    }
}
```

## strings and bytes: Text Processing Power-Ups

String and byte manipulation got major upgrades:

```go
package main

import (
    "bytes"
    "fmt"
    "strings"
)

// Builder for efficient string concatenation
func stringBuilder() {
    var sb strings.Builder
    
    // Preallocate capacity
    sb.Grow(100)
    
    // Efficient writes
    sb.WriteString("Hello")
    sb.WriteByte(' ')
    sb.WriteRune('世')
    sb.Write([]byte(" World"))
    
    // Get result (no allocation)
    result := sb.String()
    
    // Reset for reuse
    sb.Reset()
    
    // Check length and capacity
    fmt.Printf("Len: %d, Cap: %d\n", sb.Len(), sb.Cap())
}

// Cut function (Go 1.18+)
func stringCut() {
    s := "hello:world:foo"
    
    // Cut splits at first separator
    before, after, found := strings.Cut(s, ":")
    // before = "hello", after = "world:foo", found = true
    
    // Also in bytes package
    data := []byte("key=value")
    key, value, found := bytes.Cut(data, []byte("="))
}

// Clone for string safety (Go 1.18+)
func stringClone() {
    // Problem: substring keeps reference to original
    huge := strings.Repeat("x", 1<<20)  // 1MB string
    small := huge[:10]  // Still references entire 1MB!
    
    // Solution: Clone creates independent copy
    small = strings.Clone(huge[:10])  // Only 10 bytes
}

// CutPrefix and CutSuffix (Go 1.20+)
func stringCutPrefixSuffix() {
    s := "prefix_content_suffix"
    
    // Remove prefix if present
    content, found := strings.CutPrefix(s, "prefix_")
    // content = "content_suffix", found = true
    
    // Remove suffix if present
    content, found = strings.CutSuffix(content, "_suffix")
    // content = "content", found = true
    
    // Also in bytes package
    data := []byte("###data###")
    data, _ = bytes.CutPrefix(data, []byte("###"))
    data, _ = bytes.CutSuffix(data, []byte("###"))
}

// ContainsFunc for custom checking (Go 1.21+)
func stringContainsFunc() {
    s := "Hello123World"
    
    // Check if contains any digit
    hasDigit := strings.ContainsFunc(s, func(r rune) bool {
        return r >= '0' && r <= '9'
    })
    
    // IndexFunc finds first matching rune
    firstDigit := strings.IndexFunc(s, func(r rune) bool {
        return r >= '0' && r <= '9'
    })
    
    // LastIndexFunc finds last matching rune
    lastDigit := strings.LastIndexFunc(s, func(r rune) bool {
        return r >= '0' && r <= '9'
    })
}

// ReplaceAll shorthand (Go 1.12+)
func stringReplaceAll() {
    s := "foo foo foo"
    
    // Old way
    result := strings.Replace(s, "foo", "bar", -1)
    
    // New way
    result = strings.ReplaceAll(s, "foo", "bar")
}

// ToValidUTF8 for sanitization (Go 1.13+)
func stringValidUTF8() {
    // String with invalid UTF-8
    s := "Hello\xffWorld"
    
    // Replace invalid sequences
    clean := strings.ToValidUTF8(s, "�")
    // clean = "Hello�World"
}
```

## Regular Expressions: Performance Boost

The regexp package got faster and more capable:

```go
package main

import (
    "fmt"
    "regexp"
)

// Longest match mode
func regexpLongest() {
    // Default: leftmost-first (like Perl)
    re := regexp.MustCompile(`a*`)
    fmt.Println(re.FindString("aaa"))  // "aaa"
    
    // Longest: leftmost-longest
    reLongest := regexp.MustCompilePOSIX(`a*`)
    fmt.Println(reLongest.FindString("aaa"))  // "aaa" (same here, but different for complex patterns)
}

// Submatch functions
func regexpSubmatch() {
    re := regexp.MustCompile(`(\w+)@(\w+)\.(\w+)`)
    
    // Find all submatches
    matches := re.FindStringSubmatch("user@example.com")
    // matches = ["user@example.com", "user", "example", "com"]
    
    // Named groups
    reNamed := regexp.MustCompile(`(?P<user>\w+)@(?P<domain>\w+)\.(?P<tld>\w+)`)
    
    matches = reNamed.FindStringSubmatch("user@example.com")
    names := reNamed.SubexpNames()
    
    result := make(map[string]string)
    for i, name := range names {
        if i != 0 && name != "" {
            result[name] = matches[i]
        }
    }
    // result = map[user:user domain:example tld:com]
}

// ReplaceAllStringFunc for complex replacements
func regexpReplaceFunc() {
    re := regexp.MustCompile(`\b(\w+)\b`)
    
    text := "hello world foo bar"
    result := re.ReplaceAllStringFunc(text, func(match string) string {
        return strings.ToUpper(match)
    })
    // result = "HELLO WORLD FOO BAR"
    
    // With submatches
    re2 := regexp.MustCompile(`(\d+)`)
    text2 := "I have 10 apples and 20 oranges"
    
    result2 := re2.ReplaceAllStringFunc(text2, func(match string) string {
        n, _ := strconv.Atoi(match)
        return strconv.Itoa(n * 2)
    })
    // result2 = "I have 20 apples and 40 oranges"
}

// Copy for safe concurrent use
func regexpCopy() {
    re := regexp.MustCompile(`\d+`)
    
    // Create independent copy for concurrent use
    reCopy := re.Copy()
    
    // Each can be used in different goroutines
    go func() {
        reCopy.FindAllString("123 456", -1)
    }()
    
    re.FindAllString("789 012", -1)
}

// Marshaling support
func regexpMarshal() {
    type Config struct {
        Pattern *regexp.Regexp `json:"pattern"`
    }
    
    // Regexp implements encoding.TextMarshaler
    config := Config{
        Pattern: regexp.MustCompile(`\d+`),
    }
    
    data, _ := json.Marshal(config)
    // data = {"pattern":"\\d+"}
    
    var loaded Config
    json.Unmarshal(data, &loaded)
    // Pattern is restored!
}
```

## io and io/fs: Unified File Systems

The io package gained unified abstractions:

```go
package main

import (
    "embed"
    "io"
    "io/fs"
    "os"
)

//go:embed static/*
var staticFS embed.FS

// fs.FS interface - unified file system abstraction
func unifiedFS() {
    // Different FS implementations
    var filesystem fs.FS
    
    // OS file system
    filesystem = os.DirFS("/path/to/dir")
    
    // Embedded file system
    filesystem = staticFS
    
    // Sub file system
    subFS, _ := fs.Sub(filesystem, "static")
    
    // Read file from any FS
    data, err := fs.ReadFile(filesystem, "config.json")
    
    // Walk any FS
    fs.WalkDir(filesystem, ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        
        info, _ := d.Info()
        fmt.Printf("%s: %d bytes\n", path, info.Size())
        return nil
    })
    
    // Glob on any FS
    matches, _ := fs.Glob(filesystem, "*.txt")
}

// io.Discard replaces ioutil.Discard
func ioDiscard() {
    // Discard writer (like /dev/null)
    io.Copy(io.Discard, someReader)
    
    // Useful for draining readers
    io.CopyN(io.Discard, resp.Body, 1024)
}

// ReadAll moved to io
func ioReadAll() {
    // Old: ioutil.ReadAll
    // New: io.ReadAll
    data, err := io.ReadAll(reader)
    
    // With limit
    limited := io.LimitReader(reader, 1<<20)  // 1MB max
    data, err = io.ReadAll(limited)
}

// NopCloser moved to io
func ioNopCloser() {
    // Convert Reader to ReadCloser
    var reader io.Reader = strings.NewReader("data")
    var readCloser io.ReadCloser = io.NopCloser(reader)
}

// MultiReader and MultiWriter improvements
func ioMulti() {
    r1 := strings.NewReader("Hello ")
    r2 := strings.NewReader("World")
    r3 := strings.NewReader("!")
    
    // Combine multiple readers
    multi := io.MultiReader(r1, r2, r3)
    data, _ := io.ReadAll(multi)  // "Hello World!"
    
    // Multiple writers
    var buf1, buf2 bytes.Buffer
    multi2 := io.MultiWriter(&buf1, &buf2, os.Stdout)
    
    fmt.Fprintf(multi2, "Written to all")  // Written to all three
}

// Pipe improvements
func ioPipe() {
    pr, pw := io.Pipe()
    
    // Writer goroutine
    go func() {
        defer pw.Close()
        fmt.Fprintf(pw, "Hello from pipe")
    }()
    
    // Reader
    data, _ := io.ReadAll(pr)
}
```

## crypto Packages: Security Enhancements

Cryptography packages got safer and easier:

```go
package main

import (
    "crypto/rand"
    "crypto/subtle"
    "crypto/tls"
    "crypto/x509"
)

// Secure random generation
func cryptoRand() {
    // Generate random bytes
    token := make([]byte, 32)
    if _, err := rand.Read(token); err != nil {
        panic("failed to generate random token")
    }
    
    // Generate random integer
    n, err := rand.Int(rand.Reader, big.NewInt(1000000))
    
    // Generate random prime
    prime, err := rand.Prime(rand.Reader, 128)
}

// Constant-time operations
func constantTime() {
    password := []byte("secret")
    hash := []byte("hash")
    
    // Constant-time comparison (prevents timing attacks)
    if subtle.ConstantTimeCompare(password, hash) == 1 {
        fmt.Println("Match")
    }
    
    // Constant-time select
    var a, b [32]byte
    choice := 1  // 0 or 1
    subtle.ConstantTimeCopy(choice, a[:], b[:])
    
    // Constant-time byte equality
    if subtle.ConstantTimeByteEq(0x00, 0x00) == 1 {
        fmt.Println("Equal")
    }
}

// TLS 1.3 and modern cipher suites
func modernTLS() {
    config := &tls.Config{
        MinVersion: tls.VersionTLS13,  // TLS 1.3 minimum
        
        // Modern cipher suites only
        CipherSuites: []uint16{
            tls.TLS_AES_128_GCM_SHA256,
            tls.TLS_AES_256_GCM_SHA384,
            tls.TLS_CHACHA20_POLY1305_SHA256,
        },
        
        // Curve preferences
        CurvePreferences: []tls.CurveID{
            tls.X25519,
            tls.CurveP256,
        },
        
        // Session resumption
        SessionTicketsDisabled: false,
        
        // Client session cache
        ClientSessionCache: tls.NewLRUClientSessionCache(100),
    }
    
    // Dynamic certificates
    config.GetCertificate = func(info *tls.ClientHelloInfo) (*tls.Certificate, error) {
        // Return certificate based on SNI
        return getCertForDomain(info.ServerName)
    }
}

// Certificate verification
func certVerification() {
    // System root CAs
    roots, err := x509.SystemCertPool()
    
    // Add custom CA
    customCA := []byte("-----BEGIN CERTIFICATE-----...")
    roots.AppendCertsFromPEM(customCA)
    
    // Verify options
    opts := x509.VerifyOptions{
        Roots:         roots,
        Intermediates: x509.NewCertPool(),
        DNSName:       "example.com",
        CurrentTime:   time.Now(),
        KeyUsages:     []x509.ExtKeyUsage{x509.ExtKeyUsageServerAuth},
    }
    
    // Verify certificate chain
    cert, _ := x509.ParseCertificate(certDER)
    chains, err := cert.Verify(opts)
}
```

## Best Practices

### 1. Stay Current
```go
// Use latest features when they solve real problems
// But don't refactor working code without reason
```

### 2. Read Release Notes
```go
// Every Go release improves existing packages
// Check release notes for your packages
```

### 3. Benchmark Changes
```go
// New doesn't always mean faster
// Measure performance impacts
```

### 4. Gradual Adoption
```go
// Start with new code
// Migrate old code during maintenance
```

## Exercises

1. **HTTP Router**: Build a router using the new pattern matching features.

2. **JSON Stream Processor**: Create a streaming JSON processor for large files.

3. **Time Zone Handler**: Build a service that handles multiple time zones correctly.

4. **Text Processor**: Use the new string functions to build an efficient text processor.

5. **Secure Service**: Implement a service using all the modern security features.

## Summary

The enhancement of existing packages shows Go's commitment to evolution without breaking changes. Each improvement—from HTTP routing to string manipulation—makes common tasks easier while maintaining backward compatibility. These aren't just updates; they're refinements based on a decade of real-world use.

Key takeaways:
- net/http can now handle complex routing natively
- JSON handling is more flexible and performant
- Time operations are more precise and monotonic
- String/bytes operations eliminate common inefficiencies
- Security defaults have gotten stronger

Next, we'll explore the modern Go toolchain and how development tools have evolved.

---

*Continue to [Chapter 14: The Modern Go Toolchain](../part7/chapter14.md)*