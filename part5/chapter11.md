# Chapter 11: Static Analysis and Linting

## The Guardians of Code Quality

In 2015, we had `go fmt` and `go vet`. Today, Go's static analysis ecosystem rivals that of any language, with tools that catch bugs, enforce standards, detect security vulnerabilities, and even suggest optimizations. These tools don't just find problems—they teach better Go.

## Built-in Analysis Tools

### go vet: The First Line of Defense

Modern `go vet` catches far more than its 2015 ancestor:

```go
// Common issues go vet catches

// Printf format string errors
func badPrintf() {
    name := "Alice"
    age := 30
    fmt.Printf("Name: %d, Age: %s\n", name, age)  // vet: wrong types
}

// Struct tag issues
type User struct {
    Name  string `json:"name,required"`  // vet: unknown option "required"
    Email string `json:email`            // vet: missing quotes
}

// Unreachable code
func unreachable() int {
    return 42
    fmt.Println("Never executed")  // vet: unreachable code
}

// Lock/unlock mismatches
func lockIssue() {
    var mu sync.Mutex
    mu.Lock()
    defer mu.Lock()  // vet: should be Unlock
}

// Context as first parameter
func wrongContext(name string, ctx context.Context) {  // vet: context should be first
    // ...
}

// Run vet with specific checks
// $ go vet ./...
// $ go vet -printf=false ./...  // Disable specific check
```

### staticcheck: The Swiss Army Knife

Staticcheck has become the de facto standard for Go static analysis:

```bash
# Install
$ go install honnef.co/go/tools/cmd/staticcheck@latest

# Run
$ staticcheck ./...
```

```go
// Issues staticcheck catches

// Deprecated usage
func useDeprecated() {
    // SA1019: strings.Title is deprecated
    title := strings.Title("hello world")
    
    // Better: use cases.Title
    caser := cases.Title(language.English)
    title = caser.String("hello world")
}

// Inefficient string building
func inefficientString() string {
    var s string
    for i := 0; i < 1000; i++ {
        s += "x"  // SA1025: should use strings.Builder
    }
    return s
}

// Useless comparisons
func uselessComparison(x uint) {
    if x < 0 {  // SA4003: unsigned values are never < 0
        // Never happens
    }
}

// Incorrect time comparisons
func timeComparison(t1, t2 time.Time) {
    if t1 == t2 {  // SA4017: should use t1.Equal(t2)
        // Incorrect comparison
    }
}

// Inefficient slice operations
func inefficientSlice(data []byte) {
    // SA4010: this result of append is never used
    append(data, 1, 2, 3)
    
    // Should be:
    data = append(data, 1, 2, 3)
}
```

### golangci-lint: All-in-One Solution

Golangci-lint runs multiple linters in parallel:

```yaml
# .golangci.yml configuration
run:
  timeout: 5m
  tests: true
  skip-dirs:
    - vendor
    - testdata

linters:
  enable:
    - gofmt
    - govet
    - staticcheck
    - errcheck
    - gosimple
    - ineffassign
    - unused
    - gosec
    - gocritic
    - gocyclo
    - dupl
    - misspell
    - lll
    - prealloc
    - bodyclose
    - noctx
    - exhaustive
    - sqlclosecheck
    - nilerr
    - gofumpt
    - goimports
    - revive
    - durationcheck
    - wastedassign
    - importas
    - nilnil
    - stylecheck

linters-settings:
  govet:
    check-shadowing: true
  
  gocyclo:
    min-complexity: 15
  
  dupl:
    threshold: 100
  
  lll:
    line-length: 120
  
  gocritic:
    enabled-tags:
      - diagnostic
      - performance
      - style
    disabled-checks:
      - hugeParam
  
  gosec:
    excludes:
      - G104  # Ignore error checking for specific cases
  
  stylecheck:
    checks: ["all", "-ST1003"]  # All checks except one

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - dupl
        - gosec
    
    - path: migrations/
      linters:
        - gofmt
```

## Security Analysis

### gosec: Security Scanner

Find security vulnerabilities:

```go
// Issues gosec finds

// G101: Hardcoded credentials
const (
    password = "admin123"  // gosec: potential hardcoded credentials
    apiKey   = "sk-1234567890abcdef"  // gosec: potential hardcoded credentials
)

// G201: SQL injection
func sqlInjection(userInput string) {
    query := fmt.Sprintf("SELECT * FROM users WHERE name = '%s'", userInput)
    db.Query(query)  // gosec: SQL injection risk
}

// G401: Weak cryptography
func weakCrypto() {
    h := md5.New()  // gosec: MD5 is weak
    h = sha1.New()  // gosec: SHA1 is weak
}

// G104: Unhandled errors
func unhandledError() {
    file, _ := os.Open("file.txt")  // gosec: unhandled error
    defer file.Close()
}

// G304: File path traversal
func pathTraversal(userPath string) {
    data, _ := os.ReadFile(userPath)  // gosec: potential path traversal
}

// G601: Implicit memory aliasing in for loop
func memoryAliasing() {
    var wg sync.WaitGroup
    values := []string{"a", "b", "c"}
    
    for _, val := range values {
        wg.Add(1)
        go func() {
            fmt.Println(val)  // gosec: using loop variable in goroutine
            wg.Done()
        }()
    }
}

// Secure alternatives
func secureCode() {
    // Use environment variables for secrets
    password := os.Getenv("DB_PASSWORD")
    
    // Use parameterized queries
    db.Query("SELECT * FROM users WHERE name = ?", userInput)
    
    // Use strong cryptography
    h := sha256.New()
    
    // Handle errors
    file, err := os.Open("file.txt")
    if err != nil {
        return err
    }
    defer file.Close()
    
    // Validate paths
    cleanPath := filepath.Clean(userPath)
    if strings.Contains(cleanPath, "..") {
        return errors.New("invalid path")
    }
}
```

### govulncheck: Vulnerability Scanner

Check for known vulnerabilities:

```bash
# Install
$ go install golang.org/x/vuln/cmd/govulncheck@latest

# Scan for vulnerabilities
$ govulncheck ./...

# Example output:
# Scanning your code and 312 packages across 29 dependent modules
# for known vulnerabilities...
# 
# Vulnerability #1: GO-2023-1840
# HTTP/2 rapid reset can cause excessive work in net/http
# More info: https://pkg.go.dev/vuln/GO-2023-1840
# Module: golang.org/x/net
# Found in: golang.org/x/net@v0.7.0
# Fixed in: golang.org/x/net@v0.17.0
```

## Custom Static Analysis

### Writing Custom Analyzers

Create your own analysis tools:

```go
package main

import (
    "go/ast"
    "golang.org/x/tools/go/analysis"
    "golang.org/x/tools/go/analysis/singlechecker"
)

var Analyzer = &analysis.Analyzer{
    Name: "nopanic",
    Doc:  "checks for panic calls",
    Run:  run,
}

func run(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        ast.Inspect(file, func(n ast.Node) bool {
            call, ok := n.(*ast.CallExpr)
            if !ok {
                return true
            }
            
            if ident, ok := call.Fun.(*ast.Ident); ok {
                if ident.Name == "panic" {
                    pass.Reportf(ident.Pos(), 
                        "avoid using panic, return error instead")
                }
            }
            
            return true
        })
    }
    
    return nil, nil
}

func main() {
    singlechecker.Main(Analyzer)
}

// Usage:
// $ go run analyzer.go ./...
```

### Advanced Custom Analysis

```go
// Analyzer that enforces error naming conventions
var ErrorNamingAnalyzer = &analysis.Analyzer{
    Name: "errorname",
    Doc:  "checks that error variables are named 'err'",
    Run:  runErrorNaming,
}

func runErrorNaming(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        ast.Inspect(file, func(n ast.Node) bool {
            // Check variable declarations
            if decl, ok := n.(*ast.ValueSpec); ok {
                for i, name := range decl.Names {
                    if i < len(decl.Values) {
                        // Check if type is error
                        if isErrorType(pass, decl.Values[i]) {
                            if name.Name != "err" && !strings.HasSuffix(name.Name, "Err") {
                                pass.Reportf(name.Pos(), 
                                    "error variable should be named 'err' or end with 'Err', got %s", 
                                    name.Name)
                            }
                        }
                    }
                }
            }
            
            return true
        })
    }
    
    return nil, nil
}

// Analyzer for context.Context as first parameter
var ContextFirstAnalyzer = &analysis.Analyzer{
    Name: "ctxfirst",
    Doc:  "checks that context.Context is the first parameter",
    Run:  runContextFirst,
}

func runContextFirst(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        ast.Inspect(file, func(n ast.Node) bool {
            fn, ok := n.(*ast.FuncDecl)
            if !ok || fn.Type.Params == nil || len(fn.Type.Params.List) == 0 {
                return true
            }
            
            hasContext := false
            contextPosition := -1
            
            for i, param := range fn.Type.Params.List {
                if isContextType(pass, param.Type) {
                    hasContext = true
                    contextPosition = i
                    break
                }
            }
            
            if hasContext && contextPosition != 0 {
                pass.Reportf(fn.Pos(), 
                    "context.Context should be the first parameter")
            }
            
            return true
        })
    }
    
    return nil, nil
}
```

## Code Generation Analysis

### Ensuring Generated Code Quality

```go
//go:generate stringer -type=Status

type Status int

const (
    StatusPending Status = iota
    StatusActive
    StatusClosed
)

// Analyze generated code
func analyzeGenerated() {
    // Check that generated files have proper headers
    // //go:build !generate
    // // Code generated by stringer; DO NOT EDIT.
    
    // Ensure generated code passes linting
    // $ golangci-lint run --new-from-rev=HEAD~1
}
```

## Performance Analysis

### Detecting Performance Issues

```go
// Custom analyzer for performance issues
var PerformanceAnalyzer = &analysis.Analyzer{
    Name: "performance",
    Doc:  "detects common performance issues",
    Run:  runPerformance,
}

func runPerformance(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        ast.Inspect(file, func(n ast.Node) bool {
            // Detect string concatenation in loops
            if loop, ok := n.(*ast.ForStmt); ok {
                checkStringConcatInLoop(pass, loop)
            }
            
            // Detect unnecessary conversions
            if conv, ok := n.(*ast.CallExpr); ok {
                checkUnnecessaryConversion(pass, conv)
            }
            
            // Detect map lookup inefficiencies
            if ifStmt, ok := n.(*ast.IfStmt); ok {
                checkInefficient MapLookup(pass, ifStmt)
            }
            
            return true
        })
    }
    
    return nil, nil
}

// Example detections:
func performanceIssues() {
    // Detected: string concatenation in loop
    var s string
    for i := 0; i < 1000; i++ {
        s += "x"  // Should use strings.Builder
    }
    
    // Detected: unnecessary conversion
    var b []byte
    s = string(b)
    b = []byte(s)  // Unnecessary round-trip
    
    // Detected: inefficient map lookup
    m := make(map[string]int)
    if _, ok := m["key"]; ok {
        value := m["key"]  // Second lookup, should store first result
        use(value)
    }
}
```

## Integration with CI/CD

### GitHub Actions Integration

```yaml
name: Lint

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: actions/setup-go@v4
        with:
          go-version: '1.22'
      
      - name: Install tools
        run: |
          go install honnef.co/go/tools/cmd/staticcheck@latest
          go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
          go install golang.org/x/vuln/cmd/govulncheck@latest
      
      - name: Run go vet
        run: go vet ./...
      
      - name: Run staticcheck
        run: staticcheck ./...
      
      - name: Run golangci-lint
        run: golangci-lint run
      
      - name: Run govulncheck
        run: govulncheck ./...
      
      - name: Check formatting
        run: |
          if [ "$(gofmt -s -l . | wc -l)" -gt 0 ]; then
            echo "Please run 'gofmt -s -w .' to format your code"
            gofmt -s -d .
            exit 1
          fi
```

### Pre-commit Hooks

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Run linters before commit
echo "Running pre-commit checks..."

# Format code
gofmt -s -w .
goimports -w .

# Run linters
if ! go vet ./...; then
    echo "go vet failed"
    exit 1
fi

if ! staticcheck ./...; then
    echo "staticcheck failed"
    exit 1
fi

if ! golangci-lint run --fast; then
    echo "golangci-lint failed"
    exit 1
fi

echo "Pre-commit checks passed!"
```

## Automatic Fixing

### Using go fix

```go
// Old code that go fix can update
package main

import "golang.org/x/net/context"  // Old import

func oldFunction(ctx context.Context) {
    // Old API usage
}

// Run go fix
// $ go fix ./...

// After fix:
import "context"  // Updated import

func oldFunction(ctx context.Context) {
    // Updated to new API
}
```

### Auto-fixing with Tools

```bash
# Auto-fix imports
$ goimports -w .

# Auto-fix formatting
$ gofumpt -w .

# Auto-fix some staticcheck issues
$ staticcheck -fix ./...

# Auto-fix with golangci-lint
$ golangci-lint run --fix
```

## Custom Linting Rules

### Organization-Specific Rules

```go
// Custom linter for company standards
package main

import (
    "go/ast"
    "strings"
    "golang.org/x/tools/go/analysis"
)

var CompanyStandardsAnalyzer = &analysis.Analyzer{
    Name: "companystandards",
    Doc:  "enforces company coding standards",
    Run:  runCompanyStandards,
}

func runCompanyStandards(pass *analysis.Pass) (interface{}, error) {
    for _, file := range pass.Files {
        // Check package naming
        if !strings.HasPrefix(file.Name.Name, "company_") {
            pass.Reportf(file.Package, 
                "package name should start with 'company_'")
        }
        
        // Check for required comments
        ast.Inspect(file, func(n ast.Node) bool {
            if fn, ok := n.(*ast.FuncDecl); ok {
                if fn.Doc == nil && ast.IsExported(fn.Name.Name) {
                    pass.Reportf(fn.Pos(), 
                        "exported function %s needs a comment", fn.Name.Name)
                }
            }
            return true
        })
    }
    
    return nil, nil
}
```

## Gradual Adoption Strategy

### Introducing Linting to Legacy Code

```yaml
# Start permissive, gradually stricten
# Phase 1: Basic checks only
linters:
  enable:
    - gofmt
    - govet
    - ineffassign

# Phase 2: Add more linters
linters:
  enable:
    - gofmt
    - govet
    - ineffassign
    - staticcheck
    - errcheck

# Phase 3: Full suite
linters:
  enable-all: true
  disable:
    - exhaustruct  # Too strict initially
    - wrapcheck    # May be too noisy

# Use nolint comments for gradual fixes
func legacyCode() {
    //nolint:errcheck // TODO: fix in next sprint
    os.Remove("file.txt")
}
```

## Best Practices

### 1. Start Early
```yaml
# Add linting from project start
# Easier than retrofitting
```

### 2. Automate Everything
```bash
# Run on every commit
# Fix automatically where possible
```

### 3. Custom Rules for Team Standards
```go
// Encode team decisions in analyzers
```

### 4. Regular Tool Updates
```bash
# Keep tools current
go install -u honnef.co/go/tools/cmd/staticcheck@latest
```

## Exercises

1. **Custom Analyzer**: Write an analyzer that detects TODO comments and creates issues.

2. **Security Scanner**: Build a tool that finds potential security issues specific to your domain.

3. **Performance Linter**: Create a linter that detects N+1 query patterns.

4. **Migration Tool**: Write a tool that automatically migrates deprecated API usage.

5. **Metrics Collector**: Build an analyzer that collects code quality metrics.

## Summary

Static analysis has transformed from a nice-to-have to an essential part of Go development. Modern tools catch bugs that tests miss, enforce consistency that reviews might overlook, and teach best practices automatically. The key is choosing the right tools and configuring them for your team's needs.

Key takeaways:
- Start with go vet and staticcheck
- Add security scanning with gosec and govulncheck
- Use golangci-lint to run multiple tools
- Write custom analyzers for team standards
- Integrate into CI/CD pipeline

Next, we'll explore the new packages added to Go's standard library that have revolutionized how we write Go code.

---

*Continue to [Chapter 12: New Standard Packages](../part6/chapter12.md)*