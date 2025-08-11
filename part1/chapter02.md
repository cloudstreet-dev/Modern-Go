# Chapter 2: Go Modules - The Dependency Revolution

## The Dark Age of GOPATH

Before we appreciate modules, we need to remember the pain they solved. Prior to Go 1.11, every Go developer lived under the tyranny of GOPATH:

```bash
# The old way (pre-2018)
export GOPATH=$HOME/go
export PATH=$PATH:$GOPATH/bin

# All code had to live here:
$GOPATH/
├── bin/     # Compiled binaries
├── pkg/     # Compiled packages
└── src/     # ALL your source code
    ├── github.com/
    │   └── yourname/
    │       └── project/  # Your actual project, buried deep
    └── golang.org/
```

This structure meant:
- You couldn't work on Go code wherever you wanted
- Version management was "whatever's in GOPATH wins"
- Dependency conflicts were resolved by last `go get` wins
- Reproducible builds were nearly impossible
- Vendoring required third-party tools

One developer captured the frustration perfectly: "I spend more time configuring Go than writing Go."

## Enter Go Modules

Go modules, introduced experimentally in Go 1.11 and made default in Go 1.13, changed everything. A module is simply a collection of Go packages with a `go.mod` file at its root that declares the module path and its dependencies.

```bash
# The modern way (2018+)
cd ~/projects/wherever/you/want  # Work anywhere!
go mod init github.com/you/awesome-project

# Creates go.mod:
module github.com/you/awesome-project

go 1.22
```

That's it. No GOPATH. No special directories. Just code where you want it.

## Understanding go.mod

The `go.mod` file is the heart of Go modules. It's a human-readable, machine-editable manifest:

```go
module github.com/you/awesome-api

go 1.22

require (
    github.com/gorilla/mux v1.8.1
    github.com/lib/pq v1.10.9
    golang.org/x/crypto v0.17.0
)

require (
    golang.org/x/sys v0.15.0 // indirect
)

replace github.com/broken/package => github.com/fixed/package v1.2.3

exclude github.com/bad/package v1.0.0

retract (
    v1.0.0 // Published accidentally with broken API
    v1.1.0 // Security vulnerability CVE-2023-XXXX
)
```

Let's decode each directive:

### module
Declares your module's path - its name in the Go ecosystem:
```go
module github.com/you/project
```

### go
Specifies the minimum Go version required:
```go
go 1.22  // Uses new features, needs Go 1.22+

// As of Go 1.21, this also affects:
// - Language features available
// - Standard library behavior
// - Default GODEBUG settings
```

### require
Lists direct dependencies and their minimum versions:
```go
require (
    github.com/stretchr/testify v1.8.4
    golang.org/x/exp v0.0.0-20231206192017-f3f8817b8deb
)
```

### replace
Substitutes one module with another, useful for:
- Local development
- Forked dependencies
- Fixing broken upstream packages

```go
// Use local version during development
replace github.com/you/lib => ../lib

// Use fork with fixes
replace github.com/broken/pkg => github.com/you/fork v1.0.1
```

### exclude and retract
Prevent specific versions from being used:
```go
// Don't use this broken version
exclude github.com/bad/pkg v1.2.3

// Retract your own broken releases
retract [v1.0.0, v1.2.0]  // Range retraction
```

## The Companion: go.sum

Alongside `go.mod` lives `go.sum`, the cryptographic record of your dependencies:

```
github.com/gorilla/mux v1.8.1 h1:TuBL49tXwgrFYWhqrNgrUNEY92u81SPhu7sTdzQEiWY=
github.com/gorilla/mux v1.8.1/go.mod h1:AKf9I4AEqPTmMytcMc0KkNDPx6Vq5O5tNaxlg0GiMZI=
```

Each line contains:
- Module path and version
- Hash algorithm (h1 = SHA-256)
- Base64-encoded hash

This ensures your dependencies haven't been tampered with. Always commit both `go.mod` and `go.sum`.

## Essential Module Commands

### Creating and Initializing

```bash
# Start a new module
$ go mod init example.com/myproject

# Initialize module for existing code
$ cd legacy-project
$ go mod init
$ go mod tidy  # Add missing dependencies
```

### Managing Dependencies

```bash
# Add a dependency (automatic on build/test)
$ go get github.com/pkg/errors

# Add specific version
$ go get github.com/pkg/errors@v0.9.1

# Update to latest
$ go get -u github.com/pkg/errors

# Update all dependencies
$ go get -u ./...

# Remove unused dependencies
$ go mod tidy

# Download dependencies to module cache
$ go mod download
```

### Inspecting Modules

```bash
# List all dependencies
$ go list -m all

# Show available versions
$ go list -m -versions github.com/pkg/errors

# Explain why a dependency exists
$ go mod why github.com/pkg/errors

# Visualize dependency graph
$ go mod graph

# Verify dependencies haven't been tampered with
$ go mod verify
```

## Version Selection: MVS

Go uses Minimal Version Selection (MVS), a unique approach to dependency resolution. Unlike other systems that try to use the newest compatible version, Go uses the oldest version that satisfies all requirements.

```go
// Your go.mod requires:
require github.com/lib/foo v1.2.0

// A dependency requires:
require github.com/lib/foo v1.3.0

// MVS selects: v1.3.0 (minimum that satisfies both)
```

This makes builds more predictable and stable. You get newer versions only when something actually needs them.

## Working with Private Modules

Private modules require additional configuration:

```bash
# Tell Go which modules are private
$ go env -w GOPRIVATE=github.com/yourcompany/*

# Configure git for private repos
$ git config --global url."git@github.com:".insteadOf "https://github.com/"

# Use .netrc for HTTPS auth
$ cat ~/.netrc
machine github.com
  login YOUR_USERNAME
  password YOUR_TOKEN
```

For corporate environments:

```bash
# Use a private proxy
$ go env -w GOPROXY=https://proxy.company.com,direct
$ go env -w GONOSUMDB=github.com/company/*
```

## Module Proxies and Security

By default, Go uses a module proxy for public packages:

```bash
$ go env GOPROXY
https://proxy.golang.org,direct

$ go env GOSUMDB
sum.golang.org
```

The proxy provides:
- **Availability**: Modules remain available even if origin disappears
- **Security**: Checksum database prevents supply chain attacks
- **Performance**: Fast, global CDN
- **Privacy**: The proxy doesn't log personal information

Control proxy behavior:

```bash
# Bypass proxy for specific modules
$ go env -w GONOPROXY=github.com/mycompany/*

# Bypass checksum database
$ go env -w GONOSUMDB=github.com/mycompany/*

# Bypass proxy entirely (not recommended)
$ go env -w GOPROXY=direct
```

## Semantic Versioning and Module Paths

Go modules embrace semantic versioning with a twist. Major version 2+ must be included in the module path:

```go
// Version 1.x.x
module github.com/you/api

// Version 2.x.x - MUST include /v2
module github.com/you/api/v2

// Import reflects version
import "github.com/you/api/v2/client"
```

This allows multiple major versions to coexist:

```go
import (
    apiV1 "github.com/you/api/client"
    apiV2 "github.com/you/api/v2/client"
)

// Use both versions simultaneously
oldClient := apiV1.NewClient()
newClient := apiV2.NewClient()
```

## Multi-Module Repositories

Sometimes you need multiple modules in one repository:

```
myproject/
├── go.mod                 # Main module
├── cmd/
│   └── tool/
│       └── go.mod        # Separate module for tool
└── examples/
    └── go.mod            # Separate module for examples
```

Managing multi-module repos:

```bash
# Work on multiple modules together
$ go work init
$ go work use . ./cmd/tool ./examples

# Creates go.work:
go 1.22

use (
    .
    ./cmd/tool
    ./examples
)
```

## Vendoring in the Module Era

While modules reduce the need for vendoring, it's still supported:

```bash
# Create vendor directory
$ go mod vendor

# Build using vendor (not module cache)
$ go build -mod=vendor

# Verify vendor matches go.mod
$ go mod verify
```

Modern vendoring use cases:
- Hermetic builds in CI/CD
- Offline development
- Committing dependencies for audit
- Corporate policy compliance

## Module Development Workflow

Here's a typical module development workflow:

```bash
# 1. Start your project
$ mkdir awesome-tool && cd awesome-tool
$ go mod init github.com/you/awesome-tool

# 2. Write some code
$ cat > main.go << 'EOF'
package main

import (
    "log/slog"
    "github.com/spf13/cobra"
)

func main() {
    slog.Info("Starting awesome tool")
    cobra.CheckErr(rootCmd.Execute())
}
EOF

# 3. Get dependencies (happens automatically)
$ go build  # Downloads and adds to go.mod

# 4. Lock exact versions
$ go mod tidy  # Clean up and lock

# 5. Update a specific dependency
$ go get -u github.com/spf13/cobra

# 6. Test everything works
$ go test ./...

# 7. Tag a release
$ git tag v1.0.0
$ git push origin v1.0.0
```

## Debugging Module Issues

When modules misbehave, these commands help:

```bash
# See what Go is doing
$ go build -x  # Print commands
$ go get -x github.com/pkg/errors  # Debug download

# Check module resolution
$ go list -m -json github.com/pkg/errors

# Clear module cache (nuclear option)
$ go clean -modcache

# Find where modules are cached
$ go env GOMODCACHE
/Users/you/go/pkg/mod

# Debug version selection
$ go mod graph | grep package-name
$ go mod why package-name
```

## Migration Strategies

Migrating from GOPATH to modules? Here's a proven approach:

### For Simple Projects

```bash
# 1. Navigate to your project (in GOPATH)
$ cd $GOPATH/src/github.com/you/project

# 2. Initialize module
$ go mod init github.com/you/project

# 3. Build to populate dependencies
$ go build ./...

# 4. Tidy up
$ go mod tidy

# 5. Test everything
$ go test ./...

# 6. Move out of GOPATH (optional)
$ mv $GOPATH/src/github.com/you/project ~/projects/
```

### For Complex Projects

```bash
# 1. Analyze current dependencies
$ go list -json ./... | jq '.Imports, .TestImports' | sort -u

# 2. Initialize with specific version
$ go mod init github.com/you/project
$ go mod edit -go=1.19  # Match current Go version

# 3. Add known dependencies with versions
$ go get github.com/pkg/errors@v0.8.1
$ go get github.com/stretchr/testify@v1.7.0

# 4. Vendor if needed
$ go mod vendor
$ go build -mod=vendor

# 5. Gradual migration
# Keep both module and GOPATH working during transition
```

## Best Practices

### 1. Minimal Version Principle
Don't update dependencies without reason:
```bash
# Bad: Blindly update everything
$ go get -u ./...

# Good: Update specific packages when needed
$ go get -u github.com/critical/security-fix
```

### 2. Reproducible Builds
Always commit both files:
```bash
$ git add go.mod go.sum
$ git commit -m "deps: update dependencies"
```

### 3. Tool Dependencies
Track tool dependencies in your module:
```go
//go:build tools
// +build tools

package tools

import (
    _ "github.com/golangci/golangci-lint/cmd/golangci-lint"
    _ "golang.org/x/tools/cmd/stringer"
)
```

### 4. Version Tagging
Use semantic versioning for releases:
```bash
# Patch release (backwards compatible fixes)
$ git tag v1.0.1

# Minor release (backwards compatible features)
$ git tag v1.1.0

# Major release (breaking changes)
$ git tag v2.0.0  # Remember module path change!
```

### 5. Module Documentation
Document module usage in your README:
```markdown
## Installation
\```bash
go get github.com/you/awesome-tool@latest
\```

## Usage
\```go
import "github.com/you/awesome-tool"
\```
```

## Common Pitfalls and Solutions

### Pitfall 1: Forgetting go.sum
```bash
# Problem: Build fails in CI
# Solution: Always commit go.sum
$ git add go.sum
```

### Pitfall 2: Private Module Access
```bash
# Problem: Cannot download private modules
# Solution: Configure GOPRIVATE
$ go env -w GOPRIVATE=github.com/company/*
```

### Pitfall 3: Version Conflicts
```bash
# Problem: Incompatible versions
# Solution: Use MVS debugging
$ go mod graph | grep conflicting-package
$ go mod why -m conflicting-package
```

### Pitfall 4: Dirty Module Cache
```bash
# Problem: Weird build errors
# Solution: Clear and rebuild
$ go clean -modcache
$ go mod download
```

## The Future of Modules

Modules continue evolving:

- **Workspace mode** (Go 1.18+): Better multi-module development
- **Module graph pruning** (Go 1.17+): Faster downloads, smaller go.mod
- **Lazy loading** (Go 1.17+): Load only needed dependencies
- **Better error messages**: Clearer dependency conflict resolution

## Exercises

1. **Module Creation**: Create a module that imports and uses three different external packages. Tag it with version v1.0.0.

2. **Version Investigation**: For any popular package (like `github.com/gin-gonic/gin`), find:
   - Latest version
   - All available versions
   - Why it's in your module (if it is)

3. **Private Module**: Set up a private module using a GitHub private repository. Configure your environment to access it.

4. **Version Conflict**: Create two modules where:
   - Module A requires package X v1.0.0
   - Module B requires package X v2.0.0
   - A main module uses both A and B

5. **Migration Practice**: Find an old Go project using GOPATH and migrate it to modules.

## Summary

Go modules transformed Go from a language with painful dependency management to one with arguably the best dependency story in modern programming. The system is:

- **Simple**: Most of the time, it just works
- **Secure**: Cryptographic verification built-in
- **Reproducible**: Same inputs, same outputs, everywhere
- **Practical**: Designed for real-world use

The transition from GOPATH to modules represents more than a technical improvement—it's a philosophy shift. Go acknowledged that developer experience matters as much as language design.

Next, we'll explore the feature that took even longer to arrive but was even more transformative: generics. If modules freed us from dependency hell, generics freed us from `interface{}` purgatory.

---

*Continue to [Chapter 3: Understanding Type Parameters](../part2/chapter03.md)*