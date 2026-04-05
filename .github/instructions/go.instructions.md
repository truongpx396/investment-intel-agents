---
description: 'Instructions for writing Go code following idiomatic Go practices and community standards'
applyTo: '**/*.go,**/go.mod,**/go.sum'
---

# Go Development Instructions

Follow idiomatic Go practices and community standards when writing Go code. These instructions are based on [Effective Go](https://go.dev/doc/effective_go), [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments), and [Google's Go Style Guide](https://google.github.io/styleguide/go/).

## General Instructions

- Write simple, clear, and idiomatic Go code
- Favor clarity and simplicity over cleverness
- Follow the principle of least surprise
- Keep the happy path left-aligned (minimize indentation)
- Return early to reduce nesting
- Prefer early return over if-else chains; use `if condition { return }` pattern to avoid else blocks
- Make the zero value useful
- Write self-documenting code with clear, descriptive names
- Document exported types, functions, methods, and packages
- Use Go modules for dependency management
- Leverage the Go standard library instead of reinventing the wheel (e.g., use `strings.Builder` for string concatenation, `filepath.Join` for path construction)
- Prefer standard library solutions over custom implementations when functionality exists
- Write comments in English by default; translate only upon user request
- Avoid using emoji in code and comments

## Naming Conventions

### Packages

- Use lowercase, single-word package names
- Avoid underscores, hyphens, or mixedCaps
- Choose names that describe what the package provides, not what it contains
- Avoid generic names like `util`, `common`, or `base`
- Package names should be singular, not plural

#### Package Declaration Rules (CRITICAL):
- **NEVER duplicate `package` declarations** - each Go file must have exactly ONE `package` line
- When editing an existing `.go` file:
  - **PRESERVE** the existing `package` declaration - do not add another one
  - If you need to replace the entire file content, start with the existing package name
- When creating a new `.go` file:
  - **BEFORE writing any code**, check what package name other `.go` files in the same directory use
  - Use the SAME package name as existing files in that directory
  - If it's a new directory, use the directory name as the package name
  - Write **exactly one** `package <name>` line at the very top of the file
- When using file creation or replacement tools:
  - **ALWAYS verify** the target file doesn't already have a `package` declaration before adding one
  - If replacing file content, include only ONE `package` declaration in the new content
  - **NEVER** create files with multiple `package` lines or duplicate declarations

### Variables and Functions

- Use mixedCaps or MixedCaps (camelCase) rather than underscores
- Keep names short but descriptive
- Use single-letter variables only for very short scopes (like loop indices)
- Exported names start with a capital letter
- Unexported names start with a lowercase letter
- Avoid stuttering (e.g., avoid `http.HTTPServer`, prefer `http.Server`)

### Interfaces

- Name interfaces with -er suffix when possible (e.g., `Reader`, `Writer`, `Formatter`)
- Single-method interfaces should be named after the method (e.g., `Read` → `Reader`)
- Keep interfaces small and focused

### Constants

- Use MixedCaps for exported constants
- Use mixedCaps for unexported constants
- Group related constants using `const` blocks
- Consider using typed constants for better type safety

## Code Style and Formatting

### Formatting

- Always use `gofmt` to format code
- Use `goimports` to manage imports automatically
- Keep line length reasonable (no hard limit, but consider readability)
- Add blank lines to separate logical groups of code

### Comments

- Strive for self-documenting code; prefer clear variable names, function names, and code structure over comments
- Write comments only when necessary to explain complex logic, business rules, or non-obvious behavior
- Write comments in complete sentences in English by default
- Translate comments to other languages only upon specific user request
- Start sentences with the name of the thing being described
- Package comments should start with "Package [name]"
- Use line comments (`//`) for most comments
- Use block comments (`/* */`) sparingly, mainly for package documentation
- Document why, not what, unless the what is complex
- Avoid emoji in comments and code

### Error Handling

- Check errors immediately after the function call
- Don't ignore errors using `_` unless you have a good reason (document why)
- Wrap errors with context using `fmt.Errorf` with `%w` verb
- Create custom error types when you need to check for specific errors
- Place error returns as the last return value
- Name error variables `err`
- Keep error messages lowercase and don't end with punctuation

## Architecture and Project Structure

### Project Layout

This project follows Clean Architecture with a feature-based package layout. The directory structure is prescribed — do NOT deviate.

Repository root:

```
.
├── cmd/server/main.go        # Bootstrap, wiring, server start
├── config/config.go          # Env/file config struct & loader (NO client instantiation)
├── internal/                 # ALL feature application code lives here
│   └── <feature>/            # Feature modules (iam/, datastore/, audit/)
├── pkg/                      # Shared reusable code (imported by internal/ features)
│   ├── platform/             # Concrete infra clients (postgres/, redis/, otel/, logger/)
│   └── shared/               # Cross-cutting types (dto/, errors/, middleware/, model/)
├── migrations/               # Versioned SQL migration files
├── tests/                    # Contract (Bruno) and E2E tests
│   ├── contract/bruno/
│   └── e2e/
├── docs/api/                 # OpenAPI specs
├── docker-compose.yml
├── Dockerfile
├── Makefile
├── .golangci.yml
└── go.mod
```

Critical rules:
- Keep main packages in `cmd/` directory
- Group related functionality into feature packages under `internal/`
- Avoid circular dependencies
- Shared cross-cutting types live under `pkg/shared/` — NOT at the repository root or inside features

### Feature Module Layout

Each feature (e.g., `internal/iam/`, `internal/datastore/`) MUST follow this structure:

```
internal/<feature>/
├── module.go                 # DI wiring: SetupModule(appCtx)
├── model/                    # Domain entities & value objects (zero imports)
├── dto/                      # Request/response DTOs (one file per use case)
├── errors/                   # Feature-scoped error definitions
├── service/                  # Business logic (one file per use case + _test.go)
└── infra/                    # All I/O adapters
    ├── repo/
    │   └── db/               # Database repository implementations
    └── transport/
        └── http/             # HTTP handlers + controller.go (route registration)
```

`module.go` pattern: Every feature MUST expose a `SetupModule(appCtx)` function that:

- Instantiates repos/caches using platform clients from `appCtx`
- Instantiates services with those repos
- Instantiates transport controllers with those services
- Registers routes on the shared router
- Only `cmd/server/main.go` may call `SetupModule` — no other package performs wiring

Dependency direction: `Transport → Service → Repository → Model`. `Model` has zero internal imports. Violations MUST be rejected.

### Platform Infrastructure (`pkg/platform/`)

- One sub-package per technology: `postgres/`, `redis/`, `otel/`, `logger/`
- Each exposes a constructor: `New(cfg) (*Client, error)` plus health-check/shutdown
- Feature modules MUST NOT import `pkg/platform/` directly — they receive clients via `appCtx`
- Only `cmd/server/main.go` calls `pkg/platform/` constructors

### DTO Placement Rules

- Request/response DTOs (HTTP payloads, validation tags) live in the feature's `dto/` sub-package
- Transport handlers MUST convert DTOs to/from domain entities before calling services
- Domain entities in `model/` MUST NEVER carry transport concerns (no JSON tags on model types; JSON tags go on DTOs)
- Shared DTOs (e.g., pagination wrapper) live in `pkg/shared/dto/`

### Cross-Module Communication

- Feature packages MUST NOT import another feature's unexported types
- Inter-feature dependencies MUST go through exported interfaces or `pkg/shared/`
- All external service interactions (Casdoor, payment gateways, etc.) MUST be behind an interface defined in the consuming module, with concrete implementations in `infra/`

### Dependency Management

- Use Go modules (`go.mod` and `go.sum`)
- Keep dependencies minimal
- Regularly update dependencies for security patches
- Use `go mod tidy` to clean up unused dependencies
- Vendor dependencies only when necessary

## Type Safety and Language Features

### Type Definitions

- Define types to add meaning and type safety
- Use struct tags for JSON, XML, database mappings
- Prefer explicit type conversions
- Use type assertions carefully and check the second return value
- Prefer generics over unconstrained types; when an unconstrained type is truly needed, use the predeclared alias `any` instead of `interface{}` (Go 1.18+)

### Pointers vs Values

- Use pointer receivers for large structs or when you need to modify the receiver
- Use value receivers for small structs and when immutability is desired
- Use pointer parameters when you need to modify the argument or for large structs
- Use value parameters for small structs and when you want to prevent modification
- Be consistent within a type's method set
- Consider the zero value when choosing pointer vs value receivers

### Interfaces and Composition

- Accept interfaces, return concrete types
- Keep interfaces small (1-3 methods is ideal)
- Use embedding for composition
- Define interfaces close to where they're used, not where they're implemented
- Don't export interfaces unless necessary

## Concurrency

### Goroutines

- Be cautious about creating goroutines in libraries; prefer letting the caller control concurrency
- If you must create goroutines in libraries, provide clear documentation and cleanup mechanisms
- Always know how a goroutine will exit
- Use `sync.WaitGroup` or channels to wait for goroutines
- Avoid goroutine leaks by ensuring cleanup

### Channels

- Use channels to communicate between goroutines
- Don't communicate by sharing memory; share memory by communicating
- Close channels from the sender side, not the receiver
- Use buffered channels when you know the capacity
- Use `select` for non-blocking operations

### Synchronization

- Use `sync.Mutex` for protecting shared state
- Keep critical sections small
- Use `sync.RWMutex` when you have many readers
- Choose between channels and mutexes based on the use case: use channels for communication, mutexes for protecting state
- Use `sync.Once` for one-time initialization
- WaitGroup usage by Go version:
	- If `go >= 1.25` in `go.mod`, use the new `WaitGroup.Go` method ([documentation](https://pkg.go.dev/sync#WaitGroup)):
		```go
		var wg sync.WaitGroup
		wg.Go(task1)
		wg.Go(task2)
		wg.Wait()
		```
	- If `go < 1.25`, use the classic `Add`/`Done` pattern

## Error Handling Patterns

### Creating Errors

- Use `errors.New` for simple static errors
- Use `fmt.Errorf` for dynamic errors
- Create custom error types for domain-specific errors
- Export error variables for sentinel errors
- Use `errors.Is` and `errors.As` for error checking

### Error Propagation

- Add context when propagating errors up the stack
- Don't log and return errors (choose one)
- Handle errors at the appropriate level
- Consider using structured errors for better debugging

## API Design

### HTTP Handlers (Gin Framework)

This project uses **Gin** as the HTTP router/framework.

- Use `gin.HandlerFunc` for handlers; accept `*gin.Context` as the parameter
- Use Gin middleware for cross-cutting concerns (auth, CORS, request ID, logging, RBAC)
- Set appropriate status codes via `c.JSON(status, body)`, `c.AbortWithStatusJSON(status, body)`
- Handle errors gracefully and return structured JSON error responses (`{code, message, details}`)
- All routes MUST be registered under `/api/v1/` prefix (API versioning)
- Route registration happens in each feature's `infra/transport/http/controller.go`
- Group routes by feature and apply middleware per group (e.g., auth middleware, RBAC middleware with permission keys)

### JSON APIs & Error Responses

- Use struct tags to control JSON marshaling
- Validate input data
- Use pointers for optional fields
- Consider using `json.RawMessage` for delayed parsing
- Handle JSON errors appropriately

**Unified Error Schema **: ALL error responses MUST use:
```json
{ "code": "ERROR_CODE", "message": "Human-readable message", "details": [...] }
```
- Error codes come from a shared registry (`pkg/shared/errors/registry.go`)
- Use consistent HTTP status codes across all endpoints
- Error codes are stable machine keys — frontends use them for i18n translation lookup

**API Consistency Rules**:
- All timestamps MUST be ISO 8601 / RFC 3339 in UTC
- Monetary values MUST use the smallest currency unit (cents)
- Pagination MUST use a uniform scheme (`page`, `page_size`) with identical parameter names across all list endpoints
- Sorting, filtering, and search MUST follow a single convention across all endpoints

### BFF Response Shape

API responses MUST be shaped to serve the UI directly:

- **Mirror UI structure**: If the UI groups fields into sections, the response MUST reflect that grouping as nested objects
- **Consistent field naming**: A concept MUST use the same field name in every schema (e.g., `name` everywhere, never `full_name` in one and `name` in another)
- **Enum consistency**: A constrained set of values MUST carry the same `enum` definition in every schema
- **Stable unique keys**: Every list item MUST provide a stable, unique `id` or business key for frontend rendering keys
- **`key` vs `display_name` separation**: Machine identifiers (`key`) and human-readable labels (`display_name`) MUST be two distinct fields
- **Pre-computed display values**: Include pre-formatted values (e.g., `"3/5"` label) where the backend can cheaply compute them
- **No `additionalProperties` for structured data**: Use typed arrays of explicit objects for UI-rendered data

### HTTP Clients

- Keep the client struct focused on configuration and dependencies only (e.g., base URL, `*http.Client`, auth, default headers). It must not store per-request state
- Do not store or cache `*http.Request` inside the client struct, and do not persist request-specific state across calls; instead, construct a fresh request per method invocation
- Methods should accept `context.Context` and input parameters, assemble the `*http.Request` locally (or via a short-lived builder/helper created per call), then call `c.httpClient.Do(req)`
- If request-building logic is reused, factor it into unexported helper functions or a per-call builder type; never keep `http.Request` (URL params, body, headers) as fields on the long-lived client
- Ensure the underlying `*http.Client` is configured (timeouts, transport) and is safe for concurrent use; avoid mutating `Transport` after first use
- Always set headers on the request instance you’re sending, and close response bodies (`defer resp.Body.Close()`), handling errors appropriately

## Performance Optimization

### Memory Management

- Minimize allocations in hot paths
- Reuse objects when possible (consider `sync.Pool`)
- Use value receivers for small structs
- Preallocate slices when size is known
- Avoid unnecessary string conversions

### I/O: Readers and Buffers

- Most `io.Reader` streams are consumable once; reading advances state. Do not assume a reader can be re-read without special handling
- If you must read data multiple times, buffer it once and recreate readers on demand:
	- Use `io.ReadAll` (or a limited read) to obtain `[]byte`, then create fresh readers via `bytes.NewReader(buf)` or `bytes.NewBuffer(buf)` for each reuse
	- For strings, use `strings.NewReader(s)`; you can `Seek(0, io.SeekStart)` on `*bytes.Reader` to rewind
- For HTTP requests, do not reuse a consumed `req.Body`. Instead:
	- Keep the original payload as `[]byte` and set `req.Body = io.NopCloser(bytes.NewReader(buf))` before each send
	- Prefer configuring `req.GetBody` so the transport can recreate the body for redirects/retries: `req.GetBody = func() (io.ReadCloser, error) { return io.NopCloser(bytes.NewReader(buf)), nil }`
- To duplicate a stream while reading, use `io.TeeReader` (copy to a buffer while passing through) or write to multiple sinks with `io.MultiWriter`
- Reusing buffered readers: call `(*bufio.Reader).Reset(r)` to attach to a new underlying reader; do not expect it to “rewind” unless the source supports seeking
- For large payloads, avoid unbounded buffering; consider streaming, `io.LimitReader`, or on-disk temporary storage to control memory

- Use `io.Pipe` to stream without buffering the whole payload:
	- Write to `*io.PipeWriter` in a separate goroutine while the reader consumes
	- Always close the writer; use `CloseWithError(err)` on failures
	- `io.Pipe` is for streaming, not rewinding or making readers reusable

- **Warning:** When using `io.Pipe` (especially with multipart writers), all writes must be performed in strict, sequential order. Do not write concurrently or out of order—multipart boundaries and chunk order must be preserved. Out-of-order or parallel writes can corrupt the stream and result in errors.

- Streaming multipart/form-data with `io.Pipe`:
	- `pr, pw := io.Pipe()`; `mw := multipart.NewWriter(pw)`; use `pr` as the HTTP request body
	- Set `Content-Type` to `mw.FormDataContentType()`
	- In a goroutine: write all parts to `mw` in the correct order; on error `pw.CloseWithError(err)`; on success `mw.Close()` then `pw.Close()`
	- Do not store request/in-flight form state on a long-lived client; build per call
	- Streamed bodies are not rewindable; for retries/redirects, buffer small payloads or provide `GetBody`

### Profiling

- Use built-in profiling tools (`pprof`)
- Benchmark critical code paths
- Profile before optimizing
- Focus on algorithmic improvements first
- Consider using `testing.B` for benchmarks

## Testing

### TDD Workflow (NON-NEGOTIABLE)

All feature implementation MUST follow strict Test-Driven Development:

1. **Red**: Write a failing test that describes expected behavior BEFORE writing any production code
2. **Green**: Write the minimum production code to make the failing test pass
3. **Refactor**: Improve the code while keeping all tests green

- No production code change is permitted without a corresponding test change or addition
- Pull requests MUST include test commits that precede or accompany the implementation commits

### Test Organization

- Keep tests in the same package (white-box testing)
- Use `_test` package suffix for black-box testing
- Name test files with `_test.go` suffix
- Place test files next to the code they test

### Writing Tests

- **Table-driven (MANDATORY)**: All unit tests MUST use Go's table-driven pattern:
  ```go
  tests := []struct {
      name     string
      input    InputType
      expected ExpectedType
      wantErr  bool
  }{ /* cases */ }
  for _, tc := range tests {
      t.Run(tc.name, func(t *testing.T) {
          t.Parallel() // MANDATORY as first statement
          // ... test logic
      })
  }
  ```
- **`t.Parallel()` (MANDATORY)**: Every `t.Run` sub-test MUST call `t.Parallel()` as its first statement. Top-level test functions that own only parallel sub-tests SHOULD also call `t.Parallel()`.
- Name tests descriptively using `Test_functionName_scenario`
- Test both success and error cases
- Every service and repository function MUST have tests covering happy path, edge cases, and error paths
- Consider using `testify` or similar libraries when they add value, but don't over-complicate simple tests

### Test Helpers

- Mark helper functions with `t.Helper()`
- Create test fixtures for complex setup
- Use `testing.TB` interface for functions used in tests and benchmarks
- Clean up resources using `t.Cleanup()`

## Observability 

### Structured Logging

- Use **zerolog** for structured JSON logging
- Every log entry MUST include: `timestamp`, `level`, `event`, and where applicable `user_id`, `trace_id`
- Use appropriate log levels: `error` for failures, `warn` for degraded state, `info` for significant events, `debug` for development

### Tracing & Metrics

- OpenTelemetry traces MUST be wired from day one
- Prometheus-compatible metrics MUST be exposed
- Setup lives in `pkg/platform/otel/otel.go`

## Security Best Practices

### Input Validation

- Validate all external input
- Use strong typing to prevent invalid states
- Sanitize data before using in SQL queries
- Be careful with file paths from user input
- Validate and escape data for different contexts (HTML, SQL, shell)

### Cryptography

- Use standard library crypto packages
- Don't implement your own cryptography
- Use crypto/rand for random number generation
- Store passwords using bcrypt, scrypt, or argon2 (consider golang.org/x/crypto for additional options)
- Use TLS for network communication

## Documentation

### Code Documentation

- Prioritize self-documenting code through clear naming and structure
- Document all exported symbols with clear, concise explanations
- Start documentation with the symbol name
- Write documentation in English by default
- Use examples in documentation when helpful
- Keep documentation close to code
- Update documentation when code changes
- Avoid emoji in documentation and comments

### README and Documentation Files

- Include clear setup instructions
- Document dependencies and requirements
- Provide usage examples
- Document configuration options
- Include troubleshooting section

## Tools and Development Workflow

### Essential Tools

- `go fmt`: Format code
- `go vet`: Find suspicious constructs
- `golangci-lint`: Additional linting (golint is deprecated)
- `go test`: Run tests
- `go mod`: Manage dependencies
- `go generate`: Code generation

### Development Practices

- Run tests before committing
- Use pre-commit hooks for formatting and linting
- Keep commits focused and atomic
- Write meaningful commit messages
- Review diffs before committing

## Common Pitfalls to Avoid

- Not checking errors
- Ignoring race conditions
- Creating goroutine leaks
- Not using defer for cleanup
- Modifying maps concurrently
- Not understanding nil interfaces vs nil pointers
- Forgetting to close resources (files, connections)
- Using global variables unnecessarily
- Over-using unconstrained types (e.g., `any`); prefer specific types or generic type parameters with constraints. If an unconstrained type is required, use `any` rather than `interface{}`
- Not considering the zero value of types
- **Creating duplicate `package` declarations** - this is a compile error; always check existing files before adding package declarations