# Chapter 7: Embedded Files and Resources

## The End of Deployment Complexity

Remember deploying Go applications before embed? You'd carefully package your binary with configuration files, templates, static assets, and migration scripts. Docker images bloated with COPY commands. Deployment scripts that forgot files. Runtime panics when assets went missing.

Go 1.16's embed package changed everything. Now your binary IS your deployment. One file contains everything: code, assets, configurations, even entire web applications. The dream of true single-binary deployment finally became reality.

## The Pre-Embed Era

Before embed, we had creative workarounds:

```go
// Option 1: Code generation (go-bindata, packr, etc.)
//go:generate go-bindata -o assets.go templates/ static/

// Option 2: Build tags and file reading
func loadTemplate() string {
    // Works in development
    data, err := os.ReadFile("templates/index.html")
    if err != nil {
        panic("template not found - did you forget to copy it?")
    }
    return string(data)
}

// Option 3: Hard-coded strings (yes, really)
const indexHTML = `<!DOCTYPE html>
<html>
<head><title>My App</title></head>
<body><!-- Hundreds of lines... --></body>
</html>`
```

## Your First Embedded File

The embed package uses special comments to include files at compile time:

```go
package main

import (
    _ "embed"
    "fmt"
)

//go:embed hello.txt
var greeting string

//go:embed config.json
var configData []byte

func main() {
    fmt.Println(greeting)  // String contents of hello.txt
    fmt.Printf("Config size: %d bytes\n", len(configData))
}
```

Create the files:
```bash
echo "Hello from embedded file!" > hello.txt
echo '{"version": "1.0.0"}' > config.json
```

The magic happens at compile time:
```bash
$ go build -o myapp
$ rm hello.txt config.json  # Delete source files
$ ./myapp
Hello from embedded file!
Config size: 20 bytes
# Still works without source files!
```

## Embedding Patterns

### Single Files

```go
import _ "embed"

//go:embed LICENSE
var license string

//go:embed logo.png
var logoBytes []byte

//go:embed version.txt
var version string
```

### Multiple Files with embed.FS

```go
import "embed"

//go:embed templates/*.html
var templates embed.FS

//go:embed static/*
var staticFiles embed.FS

//go:embed all:migrations
var migrations embed.FS

// The "all:" prefix includes hidden files and directories
```

### Directory Trees

```go
//go:embed static
var staticFS embed.FS

// Embeds entire directory structure:
// static/
//   css/
//     main.css
//   js/
//     app.js
//   images/
//     logo.png
```

### Multiple Patterns

```go
//go:embed templates/* layouts/* components/*
var templateFS embed.FS

//go:embed config/*.yaml config/*.json
var configFS embed.FS

//go:embed *.sql
//go:embed migrations/*.sql
var sqlFiles embed.FS
```

## Working with embed.FS

The embed.FS type implements fs.FS, Go's file system interface:

```go
package main

import (
    "embed"
    "fmt"
    "io/fs"
)

//go:embed data
var dataFS embed.FS

func main() {
    // Read a file
    content, err := dataFS.ReadFile("data/config.yaml")
    if err != nil {
        panic(err)
    }
    fmt.Println(string(content))
    
    // List directory contents
    entries, err := dataFS.ReadDir("data")
    if err != nil {
        panic(err)
    }
    
    for _, entry := range entries {
        info, _ := entry.Info()
        fmt.Printf("%s (%d bytes)\n", entry.Name(), info.Size())
    }
    
    // Walk the file system
    fs.WalkDir(dataFS, ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil {
            return err
        }
        fmt.Println(path)
        return nil
    })
}
```

## Real-World Applications

### Embedding Web Applications

Build complete web applications in a single binary:

```go
package main

import (
    "embed"
    "html/template"
    "io/fs"
    "log"
    "net/http"
)

//go:embed templates/*.html
var templateFS embed.FS

//go:embed static
var staticFS embed.FS

//go:embed build
var appFS embed.FS

func main() {
    // Parse templates from embedded files
    tmpl, err := template.ParseFS(templateFS, "templates/*.html")
    if err != nil {
        log.Fatal(err)
    }
    
    // Serve static files
    staticHandler, err := fs.Sub(staticFS, "static")
    if err != nil {
        log.Fatal(err)
    }
    http.Handle("/static/", http.StripPrefix("/static/", 
        http.FileServer(http.FS(staticHandler))))
    
    // Serve SPA build
    appHandler, err := fs.Sub(appFS, "build")
    if err != nil {
        log.Fatal(err)
    }
    http.Handle("/app/", http.StripPrefix("/app/",
        http.FileServer(http.FS(appHandler))))
    
    // Dynamic routes
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        data := struct {
            Title   string
            Version string
        }{
            Title:   "My App",
            Version: getEmbeddedVersion(),
        }
        tmpl.ExecuteTemplate(w, "index.html", data)
    })
    
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}

//go:embed VERSION
var versionFile string

func getEmbeddedVersion() string {
    return strings.TrimSpace(versionFile)
}
```

### Database Migrations

Embed and run SQL migrations:

```go
package migrations

import (
    "database/sql"
    "embed"
    "fmt"
    "io/fs"
    "sort"
    "strings"
)

//go:embed *.sql
var migrationFS embed.FS

type Migration struct {
    Version string
    Name    string
    SQL     string
}

func LoadMigrations() ([]Migration, error) {
    var migrations []Migration
    
    entries, err := migrationFS.ReadDir(".")
    if err != nil {
        return nil, err
    }
    
    for _, entry := range entries {
        if !strings.HasSuffix(entry.Name(), ".sql") {
            continue
        }
        
        content, err := migrationFS.ReadFile(entry.Name())
        if err != nil {
            return nil, err
        }
        
        // Parse version from filename: 001_create_users.sql
        parts := strings.SplitN(entry.Name(), "_", 2)
        if len(parts) != 2 {
            continue
        }
        
        migrations = append(migrations, Migration{
            Version: parts[0],
            Name:    strings.TrimSuffix(parts[1], ".sql"),
            SQL:     string(content),
        })
    }
    
    // Sort by version
    sort.Slice(migrations, func(i, j int) bool {
        return migrations[i].Version < migrations[j].Version
    })
    
    return migrations, nil
}

func RunMigrations(db *sql.DB) error {
    // Create migrations table
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS migrations (
            version VARCHAR(10) PRIMARY KEY,
            name VARCHAR(255),
            applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    `)
    if err != nil {
        return fmt.Errorf("creating migrations table: %w", err)
    }
    
    migrations, err := LoadMigrations()
    if err != nil {
        return fmt.Errorf("loading migrations: %w", err)
    }
    
    for _, m := range migrations {
        // Check if already applied
        var count int
        err := db.QueryRow(
            "SELECT COUNT(*) FROM migrations WHERE version = ?",
            m.Version,
        ).Scan(&count)
        if err != nil {
            return fmt.Errorf("checking migration %s: %w", m.Version, err)
        }
        
        if count > 0 {
            continue // Already applied
        }
        
        // Run migration in transaction
        tx, err := db.Begin()
        if err != nil {
            return fmt.Errorf("beginning transaction: %w", err)
        }
        
        if _, err := tx.Exec(m.SQL); err != nil {
            tx.Rollback()
            return fmt.Errorf("running migration %s: %w", m.Version, err)
        }
        
        if _, err := tx.Exec(
            "INSERT INTO migrations (version, name) VALUES (?, ?)",
            m.Version, m.Name,
        ); err != nil {
            tx.Rollback()
            return fmt.Errorf("recording migration %s: %w", m.Version, err)
        }
        
        if err := tx.Commit(); err != nil {
            return fmt.Errorf("committing migration %s: %w", m.Version, err)
        }
        
        log.Printf("Applied migration %s: %s", m.Version, m.Name)
    }
    
    return nil
}
```

### Configuration Management

Embed environment-specific configurations:

```go
package config

import (
    "embed"
    "encoding/json"
    "fmt"
    "os"
    "gopkg.in/yaml.v3"
)

//go:embed environments/*.yaml
var configFS embed.FS

type Config struct {
    Server   ServerConfig   `yaml:"server"`
    Database DatabaseConfig `yaml:"database"`
    Features map[string]bool `yaml:"features"`
}

type ServerConfig struct {
    Port int    `yaml:"port"`
    Host string `yaml:"host"`
}

type DatabaseConfig struct {
    URL        string `yaml:"url"`
    MaxConns   int    `yaml:"max_conns"`
    MaxIdle    int    `yaml:"max_idle"`
}

func Load() (*Config, error) {
    env := os.Getenv("APP_ENV")
    if env == "" {
        env = "development"
    }
    
    // Load base configuration
    baseData, err := configFS.ReadFile("environments/base.yaml")
    if err != nil {
        return nil, fmt.Errorf("reading base config: %w", err)
    }
    
    var config Config
    if err := yaml.Unmarshal(baseData, &config); err != nil {
        return nil, fmt.Errorf("parsing base config: %w", err)
    }
    
    // Load environment-specific overrides
    envFile := fmt.Sprintf("environments/%s.yaml", env)
    if envData, err := configFS.ReadFile(envFile); err == nil {
        var overrides Config
        if err := yaml.Unmarshal(envData, &overrides); err != nil {
            return nil, fmt.Errorf("parsing %s config: %w", env, err)
        }
        mergeConfig(&config, overrides)
    }
    
    // Apply environment variables
    applyEnvVars(&config)
    
    return &config, nil
}

//go:embed schemas/*.json
var schemaFS embed.FS

func ValidateJSON(data []byte, schemaName string) error {
    schemaData, err := schemaFS.ReadFile(fmt.Sprintf("schemas/%s.json", schemaName))
    if err != nil {
        return fmt.Errorf("loading schema: %w", err)
    }
    
    // Validate using JSON schema...
    return nil
}
```

### Embedding Documentation

Include documentation with your binary:

```go
package docs

import (
    "embed"
    "fmt"
    "io/fs"
    "strings"
)

//go:embed *.md guides/*.md api/*.md
var docsFS embed.FS

func Search(query string) ([]SearchResult, error) {
    var results []SearchResult
    query = strings.ToLower(query)
    
    err := fs.WalkDir(docsFS, ".", func(path string, d fs.DirEntry, err error) error {
        if err != nil || d.IsDir() || !strings.HasSuffix(path, ".md") {
            return err
        }
        
        content, err := docsFS.ReadFile(path)
        if err != nil {
            return err
        }
        
        text := string(content)
        lowerText := strings.ToLower(text)
        
        if strings.Contains(lowerText, query) {
            // Extract title and snippet
            lines := strings.Split(text, "\n")
            title := extractTitle(lines)
            snippet := extractSnippet(lowerText, query)
            
            results = append(results, SearchResult{
                Path:    path,
                Title:   title,
                Snippet: snippet,
            })
        }
        
        return nil
    })
    
    return results, err
}

type SearchResult struct {
    Path    string
    Title   string
    Snippet string
}

func PrintHelp(topic string) error {
    helpFile := fmt.Sprintf("guides/%s.md", topic)
    content, err := docsFS.ReadFile(helpFile)
    if err != nil {
        // List available topics
        entries, _ := docsFS.ReadDir("guides")
        fmt.Println("Available help topics:")
        for _, e := range entries {
            name := strings.TrimSuffix(e.Name(), ".md")
            fmt.Printf("  - %s\n", name)
        }
        return fmt.Errorf("topic %s not found", topic)
    }
    
    fmt.Println(string(content))
    return nil
}
```

## Advanced Embedding Techniques

### Conditional Embedding

Use build tags for environment-specific embedding:

```go
//go:build production
// +build production

package assets

import "embed"

//go:embed dist/minified
var files embed.FS
```

```go
//go:build !production
// +build !production

package assets

import "embed"

//go:embed src
var files embed.FS
```

### Template Embedding with Functions

```go
package templates

import (
    "embed"
    "html/template"
    "time"
)

//go:embed *.html layouts/*.html components/*.html
var templateFS embed.FS

var funcMap = template.FuncMap{
    "formatDate": func(t time.Time) string {
        return t.Format("Jan 2, 2006")
    },
    "truncate": func(s string, n int) string {
        if len(s) <= n {
            return s
        }
        return s[:n] + "..."
    },
}

func Load() (*template.Template, error) {
    return template.New("").Funcs(funcMap).ParseFS(templateFS, 
        "*.html", "layouts/*.html", "components/*.html")
}
```

### Embedding with Compression

While embed doesn't compress files, you can embed compressed data:

```go
package assets

import (
    "bytes"
    "compress/gzip"
    "embed"
    "io"
)

//go:embed compressed/*.gz
var compressedFS embed.FS

func GetFile(name string) ([]byte, error) {
    gzName := "compressed/" + name + ".gz"
    gzData, err := compressedFS.ReadFile(gzName)
    if err != nil {
        return nil, err
    }
    
    reader, err := gzip.NewReader(bytes.NewReader(gzData))
    if err != nil {
        return nil, err
    }
    defer reader.Close()
    
    return io.ReadAll(reader)
}
```

### Virtual File System

Create a virtual file system combining embedded and real files:

```go
package vfs

import (
    "embed"
    "io/fs"
    "os"
)

//go:embed defaults
var defaultFS embed.FS

type HybridFS struct {
    embedded embed.FS
    override string  // Path to override directory
}

func New(overridePath string) *HybridFS {
    return &HybridFS{
        embedded: defaultFS,
        override: overridePath,
    }
}

func (h *HybridFS) Open(name string) (fs.File, error) {
    // Try override directory first
    if h.override != "" {
        overridePath := filepath.Join(h.override, name)
        if _, err := os.Stat(overridePath); err == nil {
            return os.Open(overridePath)
        }
    }
    
    // Fall back to embedded
    return h.embedded.Open(name)
}

func (h *HybridFS) ReadFile(name string) ([]byte, error) {
    // Try override directory first
    if h.override != "" {
        overridePath := filepath.Join(h.override, name)
        if data, err := os.ReadFile(overridePath); err == nil {
            return data, nil
        }
    }
    
    // Fall back to embedded
    return h.embedded.ReadFile(name)
}
```

## Testing with Embedded Files

### Test Fixtures

```go
package mypackage

import (
    "embed"
    "testing"
)

//go:embed testdata
var testFS embed.FS

func TestParser(t *testing.T) {
    tests := []struct {
        name     string
        fixture  string
        expected string
    }{
        {"valid JSON", "testdata/valid.json", "success"},
        {"invalid JSON", "testdata/invalid.json", "error"},
        {"empty file", "testdata/empty.json", "empty"},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            data, err := testFS.ReadFile(tt.fixture)
            if err != nil {
                t.Fatalf("reading fixture: %v", err)
            }
            
            result := Parse(data)
            if result != tt.expected {
                t.Errorf("got %v, want %v", result, tt.expected)
            }
        })
    }
}
```

### Golden Files

```go
//go:embed testdata/golden
var goldenFS embed.FS

func TestGeneration(t *testing.T) {
    input := "test input"
    got := Generate(input)
    
    golden, err := goldenFS.ReadFile("testdata/golden/output.txt")
    if err != nil {
        t.Fatal(err)
    }
    
    if !bytes.Equal(got, golden) {
        t.Errorf("output doesn't match golden file")
        
        // Write actual output for debugging
        if *updateGolden {
            os.WriteFile("testdata/golden/output.txt", got, 0644)
            t.Log("Golden file updated")
        }
    }
}
```

## Performance Considerations

### Memory Usage

Embedded files are mapped into memory:

```go
// These are different in memory usage:

//go:embed large.bin
var largeData []byte  // Entire file in memory as byte slice

//go:embed large.bin  
var largeFS embed.FS  // Memory-mapped, loaded on demand

func efficientRead() {
    // Better for large files - streaming
    file, _ := largeFS.Open("large.bin")
    defer file.Close()
    
    reader := bufio.NewReader(file)
    // Process in chunks...
}
```

### Build Size Impact

```go
// Monitor binary size with embedded assets
// Before embedding:
// $ go build -o app
// $ ls -lh app
// -rwxr-xr-x  1 user  group  2.1M  app

// After embedding 10MB of assets:
// $ go build -o app
// $ ls -lh app  
// -rwxr-xr-x  1 user  group  12.1M  app
```

### Lazy Loading Pattern

```go
package assets

import (
    "embed"
    "sync"
)

//go:embed templates
var templateFS embed.FS

var (
    tmplOnce sync.Once
    tmpl     *template.Template
    tmplErr  error
)

func GetTemplate() (*template.Template, error) {
    tmplOnce.Do(func() {
        tmpl, tmplErr = template.ParseFS(templateFS, "templates/*.html")
    })
    return tmpl, tmplErr
}
```

## Security Considerations

### Preventing Path Traversal

```go
func ServeEmbeddedFile(w http.ResponseWriter, r *http.Request) {
    // Clean the path to prevent traversal
    path := filepath.Clean(r.URL.Path)
    path = strings.TrimPrefix(path, "/")
    
    // Ensure it's within our embedded directory
    if strings.Contains(path, "..") {
        http.Error(w, "Invalid path", http.StatusBadRequest)
        return
    }
    
    data, err := staticFS.ReadFile("static/" + path)
    if err != nil {
        http.Error(w, "File not found", http.StatusNotFound)
        return
    }
    
    // Set appropriate content type
    contentType := mime.TypeByExtension(filepath.Ext(path))
    if contentType == "" {
        contentType = "application/octet-stream"
    }
    w.Header().Set("Content-Type", contentType)
    
    w.Write(data)
}
```

### Protecting Sensitive Files

```go
// Use build tags to exclude sensitive files from production

//go:build !production
// +build !production

//go:embed secrets/dev.key
var devKey []byte

//go:build production
// +build production

// In production, load from secure storage instead
var devKey []byte = loadFromVault()
```

## Best Practices

### 1. Organize Embedded Assets
```
project/
├── embed/
│   ├── static/
│   ├── templates/
│   └── migrations/
├── main.go
└── go.mod
```

### 2. Use Meaningful Variable Names
```go
// Good
//go:embed templates
var templateFS embed.FS

//go:embed static/css static/js static/images
var staticAssets embed.FS

// Bad
//go:embed stuff
var fs1 embed.FS
```

### 3. Document Embedded Content
```go
// Package assets contains all embedded static files for the web application.
// Directory structure:
//   templates/ - HTML templates
//   static/    - CSS, JS, images
//   docs/      - API documentation
package assets
```

### 4. Version Embedded Assets
```go
//go:embed VERSION
var version string

//go:embed static/app-v*.js
var versionedJS embed.FS
```

## Common Pitfalls

### Pitfall 1: Forgetting Files Exist at Compile Time

```go
// This won't work - embed happens at compile time
filename := "dynamic.txt"
//go:embed filename  // Error: can't embed variable
```

### Pitfall 2: Embedding Outside Module

```go
// Can't embed files outside module root
//go:embed ../../../etc/passwd  // Error: outside module
```

### Pitfall 3: Large Binary Sizes

```go
// Be careful with large embeddings
//go:embed videos/*.mp4  // Binary could be gigabytes!
```

## Exercises

1. **Static Site Generator**: Build a static site generator that embeds templates and outputs HTML.

2. **Self-Extracting Archive**: Create a program that embeds files and can extract them on demand.

3. **Embedded Database**: Implement a simple database that stores its data in embedded files.

4. **Plugin System**: Design a plugin system using embedded Go files.

5. **Asset Pipeline**: Build an asset pipeline that embeds and serves optimized web assets.

## Summary

The embed package transformed Go deployment from a multi-file dance to a single binary drop. It's not just about convenience—it's about reliability, security, and simplicity. Your binary becomes self-contained, version-consistent, and deployment-proof.

Key takeaways:
- Embed at compile time with `//go:embed` directives
- Use embed.FS for file systems, strings/bytes for single files
- Embedded files become part of your binary
- Perfect for templates, migrations, static assets, and configurations
- No runtime dependencies, no missing files

Next, we'll explore how Go's performance has evolved, from compiler optimizations to runtime improvements that make modern Go faster than ever.

---

*Continue to [Chapter 8: Modern Go Performance](../part4/chapter08.md)*