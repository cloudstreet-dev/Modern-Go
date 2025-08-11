# Chapter 15: Development Tools

## The Developer's Workshop

Beyond the core toolchain lies a rich ecosystem of development tools that make Go development a joy. From intelligent IDEs to powerful debuggers, from code generators to documentation tools, the modern Go developer has access to tools that would have seemed magical in 2015. This chapter explores the essential tools that define modern Go development.

## gopls: The Language Server Protocol

The Go language server powers intelligent IDE features:

### Installation and Configuration

```bash
# Install latest gopls
$ go install golang.org/x/tools/gopls@latest

# Check version
$ gopls version

# Generate config
$ gopls config generate > ~/.config/gopls/config.yaml
```

### Configuration Options

```yaml
# gopls configuration
formatting:
  gofumpt: true
  local: "github.com/mycompany"
  
ui:
  documentation:
    hoverKind: "FullDocumentation"
    linkTarget: "pkg.go.dev"
    linksInHover: true
  
  completion:
    usePlaceholders: true
    completionBudget: 100ms
    matcher: "fuzzy"
    
  diagnostic:
    analyses:
      nilness: true
      unusedparams: true
      unusedwrite: true
      useany: true
    annotations:
      bounds: true
      escape: true
      inline: true
      nil: true
    
  codelenses:
    gc_details: true
    generate: true
    regenerate_cgo: true
    tidy: true
    upgrade_dependency: true
    vendor: true
    
  semanticTokens: true
  noSemanticString: false
```

### VS Code Integration

```json
// settings.json
{
  "go.useLanguageServer": true,
  "go.languageServerFlags": [
    "-rpc.trace",
    "serve",
    "--debug=localhost:6060"
  ],
  "gopls": {
    "ui.semanticTokens": true,
    "ui.completion.usePlaceholders": true,
    "formatting.gofumpt": true,
    "ui.diagnostic.staticcheck": true,
    "ui.documentation.linkTarget": "pkg.go.dev"
  },
  "go.formatTool": "gofumpt",
  "go.lintTool": "golangci-lint",
  "go.lintFlags": ["--fast"],
  "go.testFlags": ["-v", "-race"],
  "go.testTimeout": "30s",
  "go.coverOnSave": true,
  "go.coverageDecorator": {
    "type": "gutter",
    "coveredHighlightColor": "rgba(64,128,64,0.5)",
    "uncoveredHighlightColor": "rgba(128,64,64,0.5)",
    "coveredGutterStyle": "blockgreen",
    "uncoveredGutterStyle": "blockred"
  }
}
```

## Delve: Advanced Debugging

Delve is the standard Go debugger:

### Basic Debugging

```bash
# Debug a program
$ dlv debug main.go

# Debug a test
$ dlv test

# Attach to running process
$ dlv attach <pid>

# Debug core dump
$ dlv core <executable> <core>

# Remote debugging
$ dlv debug --headless --listen=:2345 --api-version=2
```

### Debugging Commands

```bash
(dlv) break main.main           # Set breakpoint
(dlv) break file.go:42          # Break at line
(dlv) condition 1 i > 10        # Conditional breakpoint
(dlv) continue                  # Run to breakpoint
(dlv) next                      # Step over
(dlv) step                      # Step into
(dlv) stepout                   # Step out
(dlv) print variable            # Print variable
(dlv) locals                    # Show local variables
(dlv) args                      # Show function arguments
(dlv) stack                     # Show stack trace
(dlv) goroutines                # List goroutines
(dlv) goroutine 5               # Switch to goroutine
(dlv) frame 2                   # Switch stack frame
(dlv) list                      # Show source code
(dlv) disass                    # Show assembly
(dlv) regs                      # Show registers
(dlv) watch variable            # Set watchpoint
(dlv) display -a variable       # Auto-display on stop
```

### Debugging Configuration

```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Launch Package",
      "type": "go",
      "request": "launch",
      "mode": "auto",
      "program": "${workspaceFolder}",
      "env": {
        "DEBUG": "true"
      },
      "args": ["--config", "debug.yaml"],
      "showLog": true,
      "trace": "verbose"
    },
    {
      "name": "Attach to Process",
      "type": "go",
      "request": "attach",
      "mode": "local",
      "processId": "${command:pickProcess}"
    },
    {
      "name": "Debug Test",
      "type": "go",
      "request": "launch",
      "mode": "test",
      "program": "${workspaceFolder}",
      "args": ["-test.run", "TestSpecific"]
    }
  ]
}
```

## Code Generation Tools

### Stringer

Generate String() methods for constants:

```go
//go:generate stringer -type=Status -linecomment
type Status int

const (
    StatusPending Status = iota // pending
    StatusActive                // active
    StatusClosed                // closed
)
```

### Mock Generation

```bash
# mockgen for interfaces
$ go install github.com/golang/mock/mockgen@latest
$ mockgen -source=interface.go -destination=mock_interface.go

# mockery - more features
$ go install github.com/vektra/mockery/v2@latest
$ mockery --all --output=mocks
```

### SQL Code Generation

```bash
# sqlc - compile SQL to type-safe Go
$ sqlc generate

# sqlboiler - ORM generator
$ sqlboiler mysql

# ent - entity framework
$ go run -mod=mod entgo.io/ent/cmd/ent generate ./ent/schema
```

### Protocol Buffers

```bash
# Install protoc plugins
$ go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
$ go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Generate code
$ protoc --go_out=. --go-grpc_out=. api.proto
```

## Documentation Tools

### pkgsite: Local Documentation Server

```bash
# Install pkgsite
$ go install golang.org/x/pkgsite/cmd/pkgsite@latest

# Serve local documentation
$ pkgsite -http=:8080

# Open in browser
$ open http://localhost:8080/github.com/user/project
```

### godoc: Classic Documentation

```bash
# Install godoc
$ go install golang.org/x/tools/cmd/godoc@latest

# Serve documentation
$ godoc -http=:6060

# Generate static HTML
$ godoc -url="/pkg/github.com/user/project" > doc.html
```

### Documentation Comments

```go
// Package mathutil provides mathematical utility functions.
//
// The package focuses on performance-critical operations
// commonly needed in scientific computing.
//
// # Basic Usage
//
// Import the package:
//
//	import "github.com/example/mathutil"
//
// Use the functions:
//
//	result := mathutil.FastSqrt(16.0)
//
// # Performance
//
// All functions are optimized for speed over precision.
// For high-precision needs, use the standard math package.
//
// [math]: https://pkg.go.dev/math
// [benchmarks]: https://github.com/example/mathutil/blob/main/BENCHMARKS.md
package mathutil

// FastSqrt computes an approximate square root using Newton's method.
//
// It trades precision for speed, achieving 2-3x performance improvement
// over [math.Sqrt] for typical inputs.
//
// Example:
//
//	x := FastSqrt(16.0)  // Returns approximately 4.0
//
// The function panics if x is negative.
func FastSqrt(x float64) float64 {
    if x < 0 {
        panic("negative input")
    }
    // Implementation...
}
```

## Project Templates and Generators

### Project Layout Tools

```bash
# gonew - official project generator
$ go install golang.org/x/tools/cmd/gonew@latest
$ gonew github.com/user/template github.com/user/newproject

# cobra-cli for CLI apps
$ go install github.com/spf13/cobra-cli@latest
$ cobra-cli init
$ cobra-cli add serve
$ cobra-cli add config

# buf for Protocol Buffers
$ buf generate
$ buf lint
$ buf breaking --against '.git#branch=main'
```

### Standard Project Layout

```
project/
├── cmd/                # Main applications
│   ├── api/
│   └── cli/
├── internal/           # Private packages
│   ├── config/
│   ├── database/
│   └── service/
├── pkg/               # Public packages
│   └── client/
├── api/               # API definitions
│   └── v1/
├── web/               # Web assets
│   ├── static/
│   └── templates/
├── scripts/           # Build/install scripts
├── build/             # Docker, CI configs
├── deployments/       # Deployment configs
├── test/              # Additional test apps/data
├── docs/              # Documentation
├── tools/             # Supporting tools
├── examples/          # Example code
├── .github/           # GitHub specific
│   └── workflows/
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── README.md
```

## Task Runners and Build Tools

### Makefiles for Go

```makefile
# Modern Go Makefile
.PHONY: all build test clean

BINARY_NAME=myapp
DOCKER_IMAGE=myapp:latest
GO_FILES=$(shell find . -name '*.go' -type f)

all: test build

build:
	go build -v -o bin/$(BINARY_NAME) ./cmd/api

test:
	go test -v -race -coverprofile=coverage.out ./...
	go tool cover -html=coverage.out -o coverage.html

bench:
	go test -bench=. -benchmem ./...

lint:
	golangci-lint run
	gosec ./...

fmt:
	gofumpt -l -w .
	goimports -w .

generate:
	go generate ./...

docker:
	docker build -t $(DOCKER_IMAGE) .

run:
	go run ./cmd/api

install-tools:
	go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
	go install mvdan.cc/gofumpt@latest
	go install golang.org/x/tools/cmd/goimports@latest

clean:
	go clean
	rm -rf bin/ coverage.*

deps:
	go mod download
	go mod tidy

upgrade:
	go get -u ./...
	go mod tidy
```

### Task (Taskfile)

```yaml
# Taskfile.yml
version: '3'

vars:
  BINARY: myapp
  
tasks:
  default:
    cmds:
      - task: test
      - task: build
      
  build:
    desc: Build the application
    cmds:
      - go build -o bin/{{.BINARY}} ./cmd/api
    sources:
      - ./**/*.go
    generates:
      - bin/{{.BINARY}}
      
  test:
    desc: Run tests
    cmds:
      - go test -race ./...
      
  watch:
    desc: Watch for changes
    cmds:
      - modd
```

## Performance Analysis Tools

### Benchstat

```bash
# Install benchstat
$ go install golang.org/x/perf/cmd/benchstat@latest

# Compare benchmarks
$ go test -bench=. -count=10 > old.txt
# make changes
$ go test -bench=. -count=10 > new.txt
$ benchstat old.txt new.txt

# Output:
name     old time/op  new time/op  delta
Parse-8  1.30µs ± 2%  0.95µs ± 1%  -26.92%  (p=0.000 n=10+10)
```

### go-torch (Flame Graphs)

```bash
# Generate flame graph
$ go test -bench=. -cpuprofile=cpu.prof
$ go tool pprof -http=:8080 cpu.prof
# Opens interactive flame graph in browser
```

## Database Tools

### Database Migration Tools

```bash
# golang-migrate
$ migrate create -ext sql -dir migrations create_users_table
$ migrate -database postgres://... up
$ migrate -database postgres://... down

# goose
$ goose create add_users sql
$ goose up
$ goose down
```

### Database Drivers and ORMs

```go
// pgx - PostgreSQL driver
import "github.com/jackc/pgx/v5"

// GORM - ORM
import "gorm.io/gorm"

// sqlx - extensions to database/sql
import "github.com/jmoiron/sqlx"

// Bun - lightweight ORM
import "github.com/uptrace/bun"
```

## API Development Tools

### OpenAPI/Swagger

```bash
# swag - generate from annotations
$ swag init -g main.go

# oapi-codegen - generate from OpenAPI spec
$ oapi-codegen -package api spec.yaml > api.go
```

### GraphQL

```bash
# gqlgen - GraphQL server generator
$ go run github.com/99designs/gqlgen init
$ go run github.com/99designs/gqlgen generate
```

### gRPC

```bash
# Evans - gRPC client
$ evans -r repl

# grpcurl - curl for gRPC
$ grpcurl -plaintext localhost:50051 list
$ grpcurl -plaintext -d '{"name": "World"}' localhost:50051 helloworld.Greeter/SayHello
```

## Continuous Integration Tools

### golangci-lint in CI

```yaml
# .golangci.yml
run:
  timeout: 5m
  issues-exit-code: 1
  tests: true

output:
  format: colored-line-number
  print-issued-lines: true
  print-linter-name: true

linters-settings:
  errcheck:
    check-type-assertions: true
  govet:
    check-shadowing: true
  gocyclo:
    min-complexity: 15
  dupl:
    threshold: 100
  goconst:
    min-len: 2
    min-occurrences: 2

linters:
  enable-all: true
  disable:
    - exhaustruct
    - depguard

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - dupl
        - gosec
```

## IDE Configurations

### JetBrains GoLand Settings

```xml
<!-- .idea/watcherTasks.xml -->
<TaskOptions>
  <option name="arguments" value="fmt $FilePath$" />
  <option name="command" value="gofumpt" />
  <option name="runOnExternalChanges" value="false" />
</TaskOptions>
```

### Vim/Neovim Configuration

```vim
" init.vim or .vimrc
Plug 'fatih/vim-go', { 'do': ':GoUpdateBinaries' }

let g:go_fmt_command = "gofumpt"
let g:go_metalinter_enabled = ['vet', 'golint', 'errcheck']
let g:go_metalinter_autosave = 1
let g:go_auto_type_info = 1
let g:go_def_mode='gopls'
let g:go_info_mode='gopls'
```

## Best Practices

### 1. Tool Versioning
```go
//go:build tools
// +build tools

package tools

import (
    _ "github.com/golangci/golangci-lint/cmd/golangci-lint"
    _ "golang.org/x/tools/cmd/stringer"
)
```

### 2. Pre-commit Hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/golangci/golangci-lint
    rev: v1.54.2
    hooks:
      - id: golangci-lint
```

### 3. Editor Configuration
```
# .editorconfig
root = true

[*.go]
indent_style = tab
indent_size = 4
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true
```

### 4. Tool Configuration
```json
// .vscode/extensions.json
{
  "recommendations": [
    "golang.go",
    "ms-vscode.vscode-typescript-tslint-plugin",
    "zxh404.vscode-proto3"
  ]
}
```

## Exercises

1. **Custom Code Generator**: Build a code generator that creates CRUD operations from struct definitions.

2. **Development Environment**: Set up a complete Go development environment with all modern tools.

3. **CI/CD Pipeline**: Create a comprehensive CI/CD pipeline with linting, testing, and deployment.

4. **Documentation Site**: Generate and host documentation for a Go project.

5. **Performance Analysis**: Profile and optimize a Go application using modern tools.

## Summary

The modern Go development toolkit extends far beyond the compiler. From intelligent language servers to sophisticated debuggers, from code generators to documentation tools, today's Go developer has access to a rich ecosystem that accelerates development and improves code quality. The key is choosing the right tools for your workflow and keeping them properly configured and up to date.

Key takeaways:
- gopls provides IDE intelligence for any editor
- Delve enables sophisticated debugging
- Code generation reduces boilerplate
- Documentation tools make sharing knowledge easy
- Task runners and build tools streamline workflows

Next, we'll explore building modern web services with these tools and techniques.

---

*Continue to [Chapter 16: Building Modern Web Services](../part8/chapter16.md)*