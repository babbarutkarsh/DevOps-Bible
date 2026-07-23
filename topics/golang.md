---
title: Go (Golang)
nav_order: 70
description: "Go (Golang) — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Go (Golang) for DevOps — Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

**Table of Contents:**
1. [Why Go for DevOps](#why-go-for-devops)
2. [Go Fundamentals](#go-fundamentals)
3. [Concurrency Deep Dive](#concurrency-deep-dive)
4. [Concurrency Patterns](#concurrency-patterns)
5. [Sync Primitives](#sync-primitives)
6. [Race Conditions & Detection](#race-conditions--detection)
7. [Memory & Performance](#memory--performance)
8. [Slices, Arrays & Maps](#slices-arrays--maps)
9. [Interfaces & Type System](#interfaces--type-system)
10. [Error Handling](#error-handling)
11. [Defer, Panic, Recover](#defer-panic-recover)
12. [HTTP Server & Client](#http-server--client)
13. [CLI Tools](#cli-tools)
14. [Testing & Benchmarking](#testing--benchmarking)
15. [Modules & Dependencies](#modules--dependencies)
16. [Building for Containers](#building-for-containers)
17. [Common Gotchas](#common-gotchas)
18. [Coding Interview Questions](#coding-interview-questions)
19. [Key Resources](#key-resources)

---

## Why Go for DevOps

| Feature | Why It Matters |
|---------|---------------|
| **Static binary** | No runtime deps — perfect for containers |
| **Fast compilation** | Quick CI builds, instant feedback |
| **Concurrency** | Goroutines for parallel operations (M:N scheduling) |
| **Strong stdlib** | HTTP, JSON, crypto, OS interaction built-in |
| **Cross-compilation** | `GOOS=linux GOARCH=amd64 go build` — trivial |
| **Small footprint** | Minimal Docker images (scratch, distroless) |
| **Simple syntax** | Fast onboarding, readable code |

**Tools Written in Go:** Kubernetes, Docker, Terraform, Prometheus, Helm, Istio, Vault, ArgoCD, etcd, Traefik, Cilium, containerd, runc, Flux, Jaeger, CockroachDB, InfluxDB, Consul, Nomad, Packer, Minio, Telegraf

---

## Go Fundamentals

### 🔥 Q: Explain Go's type system — structs, methods, and pointer vs value receivers.

```go
type Server struct {
    Name string `json:"name"` // struct tags for JSON marshaling
    IP   string `json:"ip"`
    Port int    `json:"port"`
}

// Value receiver — works on a copy
func (s Server) Address() string {
    return fmt.Sprintf("%s:%d", s.IP, s.Port)
}

// Pointer receiver — modifies the original
func (s *Server) UpdatePort(port int) {
    s.Port = port // mutates the receiver
}
```

**When to use pointer receivers:**
- Method needs to modify the receiver
- Receiver is large (avoid copying)
- Consistency: if one method uses `*T`, all should

### 🔥 Q: How do interfaces work in Go?

```go
// Interfaces are satisfied implicitly — no "implements" keyword
type HealthChecker interface {
    CheckHealth() error
}

type HTTPServer struct{ URL string }
func (h HTTPServer) CheckHealth() error {
    resp, err := http.Get(h.URL + "/health")
    if err != nil { return err }
    defer resp.Body.Close()
    if resp.StatusCode != 200 { return fmt.Errorf("unhealthy: %d", resp.StatusCode) }
    return nil
}
// HTTPServer implicitly satisfies HealthChecker

// Empty interface — accepts any type
func logValue(v interface{}) { fmt.Println(v) }
// Go 1.18+: use `any` instead of `interface{}`
func logValue2(v any) { fmt.Println(v) }
```

**Interface gotcha:** A nil pointer stored in an interface is NOT a nil interface.

```go
var h *HTTPServer = nil
var checker HealthChecker = h
fmt.Println(checker == nil) // false! (type is non-nil)
```

---

## Concurrency Deep Dive

### 🔥 Q: Explain Go's concurrency model — goroutines vs OS threads.

**Go uses M:N scheduling (the GMP model):**
- **G (goroutine):** Lightweight user-space thread (~2KB stack, grows dynamically)
- **M (machine):** OS thread
- **P (processor):** Logical CPU, holds run queue of goroutines

**M:N:** Many goroutines (G) are multiplexed onto fewer OS threads (M) by the runtime scheduler. Default `GOMAXPROCS` = number of CPU cores.

```
┌────────┐   ┌────────┐   ┌────────┐
│   P1   │   │   P2   │   │   P3   │
│ [G G G]│   │ [G G] │   │  [G]  │
└────┬───┘   └────┬───┘   └────┬───┘
     │            │            │
     v            v            v
   [M1]         [M2]         [M3]  <-- OS threads
```

**Why goroutines are cheap:**
- Small initial stack (OS threads: 1-2MB)
- Fast context switching (no syscall)
- Can spawn millions

### 🔥 Q: Buffered vs unbuffered channels?

```go
// Unbuffered — sender blocks until receiver reads
ch := make(chan int)
go func() { ch <- 42 }() // blocks until someone reads
val := <-ch

// Buffered — sender blocks only when buffer is full
ch := make(chan int, 3)
ch <- 1 // non-blocking
ch <- 2 // non-blocking
ch <- 3 // non-blocking
ch <- 4 // blocks until someone reads
```

### ⭐ Q: What happens when you send/receive on a closed channel?

```go
ch := make(chan int, 2)
ch <- 1
close(ch)

// Reading from closed channel returns zero value + false
val, ok := <-ch  // val=1, ok=true
val, ok = <-ch   // val=0, ok=false (closed)

// Sending on closed channel panics
ch <- 2 // panic: send on closed channel
```

**Best practice:** Only the sender should close a channel.

### 💡 Q: What's a nil channel?

```go
var ch chan int // nil channel
// Sending or receiving on nil channel blocks forever
ch <- 1  // deadlock
<-ch     // deadlock

// Use case: disable a case in select
select {
case <-ch: // never fires if ch is nil
case <-time.After(1*time.Second):
    fmt.Println("timeout")
}
```

### 🔥 Q: Explain `select` with channels.

```go
select {
case msg := <-ch1:
    fmt.Println("received from ch1:", msg)
case ch2 <- value:
    fmt.Println("sent to ch2")
case <-time.After(5 * time.Second):
    fmt.Println("timeout")
default: // non-blocking, runs if no other case is ready
    fmt.Println("no activity")
}
```

- `select` blocks until one case is ready
- If multiple cases are ready, picks one at random
- `default` makes it non-blocking

---

## Concurrency Patterns

### 🔥 Q: Implement a worker pool pattern.

```go
func workerPool(jobs <-chan int, results chan<- int, numWorkers int) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                fmt.Printf("Worker %d processing job %d\n", workerID, job)
                results <- job * 2 // simulate work
            }
        }(i)
    }
    wg.Wait()
    close(results)
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)

    go workerPool(jobs, results, 3)

    // Send jobs
    for i := 1; i <= 10; i++ {
        jobs <- i
    }
    close(jobs)

    // Collect results
    for r := range results {
        fmt.Println("Result:", r)
    }
}
```

### ⭐ Q: Fan-out / fan-in pattern?

```go
// Fan-out: distribute work to multiple goroutines
func fanOut(input <-chan int, numWorkers int) []<-chan int {
    outputs := make([]<-chan int, numWorkers)
    for i := 0; i < numWorkers; i++ {
        out := make(chan int)
        outputs[i] = out
        go func(ch chan int) {
            defer close(ch)
            for val := range input {
                ch <- val * 2 // process
            }
        }(out)
    }
    return outputs
}

// Fan-in: merge multiple channels into one
func fanIn(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    output := func(ch <-chan int) {
        defer wg.Done()
        for val := range ch {
            merged <- val
        }
    }

    wg.Add(len(channels))
    for _, ch := range channels {
        go output(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

### ⭐ Q: Pipeline pattern?

```go
// Stage 1: generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: square numbers
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: sum
func sum(in <-chan int) int {
    total := 0
    for n := range in {
        total += n
    }
    return total
}

func main() {
    // Pipeline: generate → square → sum
    nums := generate(1, 2, 3, 4, 5)
    squared := square(nums)
    result := sum(squared)
    fmt.Println(result) // 55
}
```

### 🔥 Q: When and how to use `context` for cancellation?

```go
func processWithTimeout(ctx context.Context, url string) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    client := &http.Client{}
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    // Process response...
    return nil
}

func main() {
    // Timeout context
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := processWithTimeout(ctx, "https://example.com"); err != nil {
        fmt.Println("Error:", err) // "context deadline exceeded" after 5s
    }
}

// Cancellation signal
ctx, cancel := context.WithCancel(context.Background())
go func() {
    time.Sleep(2 * time.Second)
    cancel() // trigger cancellation
}()

select {
case <-ctx.Done():
    fmt.Println("Cancelled:", ctx.Err())
}
```

**Context best practices:**
- Pass `context.Context` as first parameter to functions
- Never store context in a struct
- Use `context.WithValue` sparingly (request-scoped values like trace IDs)

---

## Sync Primitives

### 🔥 Q: Mutex vs RWMutex?

```go
// Mutex — exclusive lock
var mu sync.Mutex
var counter int
mu.Lock()
counter++
mu.Unlock()

// RWMutex — multiple readers OR one writer
var rwmu sync.RWMutex
var data map[string]int

// Read lock (multiple goroutines can hold)
rwmu.RLock()
val := data["key"]
rwmu.RUnlock()

// Write lock (exclusive)
rwmu.Lock()
data["key"] = 42
rwmu.Unlock()
```

**When to use RWMutex:** Read-heavy workloads (e.g., config cache).

### 🔥 Q: WaitGroup usage?

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Println("Worker", id)
    }(i)
}

wg.Wait() // blocks until all Done() called
```

**Common mistake:** Calling `Add` inside the goroutine (race condition).

### ⭐ Q: sync.Once — when and why?

```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{} // runs exactly once, even with concurrency
    })
    return instance
}
```

**Use case:** Lazy initialization (DB connection, config loading).

### ⭐ Q: Atomic operations?

```go
import "sync/atomic"

var counter int64

atomic.AddInt64(&counter, 1)
val := atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, 100)
atomic.CompareAndSwapInt64(&counter, 100, 200)
```

**When to use:** Lock-free counters, flags. Faster than mutex for simple types.

---

## Race Conditions & Detection

### 🔥 Q: What's a data race?

**Data race:** Two goroutines access the same variable concurrently, and at least one is a write.

```go
// RACE CONDITION
var counter int
for i := 0; i < 1000; i++ {
    go func() { counter++ }() // NOT safe
}

// FIX 1: Mutex
var mu sync.Mutex
for i := 0; i < 1000; i++ {
    go func() {
        mu.Lock()
        counter++
        mu.Unlock()
    }()
}

// FIX 2: Atomic
var counter int64
for i := 0; i < 1000; i++ {
    go func() { atomic.AddInt64(&counter, 1) }()
}
```

### 🔥 Q: How to detect races?

```bash
# Build and run with race detector
go run -race main.go
go test -race ./...
go build -race

# Output when race detected:
# WARNING: DATA RACE
# Write at 0x00c000018090 by goroutine 7:
#   main.increment()
# Previous read at 0x00c000018090 by goroutine 6:
#   main.read()
```

**Production:** Race detector adds ~10x overhead — only for testing/staging.

### ⭐ Q: Common race pitfalls?

**Loop variable capture (pre-Go 1.22):**

```go
// Before Go 1.22 — RACE
urls := []string{"url1", "url2", "url3"}
for _, url := range urls {
    go func() {
        fetch(url) // all goroutines see "url3"
    }()
}

// Fix: pass as argument
for _, url := range urls {
    go func(u string) {
        fetch(u)
    }(url)
}

// Go 1.22+: loop variables are now per-iteration (no capture bug)
```

**Shared map without locking:**

```go
// RACE
m := make(map[string]int)
go func() { m["a"] = 1 }()
go func() { m["b"] = 2 }()

// FIX: Use sync.Mutex or sync.Map
var mu sync.Mutex
go func() { mu.Lock(); m["a"] = 1; mu.Unlock() }()
```

---

## Memory & Performance

### ⭐ Q: How does Go's garbage collector work?

- **Concurrent mark-and-sweep** (tri-color marking)
- **Low-latency:** Sub-millisecond pauses in Go 1.19+
- **Tuning knobs:**
  - `GOGC=100` (default): GC when heap grows 100% since last GC
  - `GOMEMLIMIT=4GiB` (Go 1.19+): soft memory limit

```bash
GOGC=200 ./myapp  # less frequent GC, higher memory
GOMEMLIMIT=2GiB ./myapp  # cap memory usage
```

### 💡 Q: Escape analysis — stack vs heap?

```go
// Stays on stack (fast)
func localVar() int {
    x := 42
    return x
}

// Escapes to heap (GC pressure)
func escapePointer() *int {
    x := 42
    return &x // x escapes (pointer outlives function)
}

// Check with:
go build -gcflags="-m" main.go
# Output: ./main.go:7:2: moved to heap: x
```

**Avoid heap allocations in hot paths:** Return values, not pointers, when possible.

### ⭐ Q: Profiling Go apps?

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    // Your app...
}
```

```bash
# CPU profile
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile
go tool pprof http://localhost:6060/debug/pprof/heap

# Goroutine count
curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

---

## Slices, Arrays & Maps

### 🔥 Q: Slice vs array?

```go
// Array — fixed size, value type
var arr [3]int = [3]int{1, 2, 3}
arr2 := arr // copies all elements

// Slice — dynamic size, reference to underlying array
slice := []int{1, 2, 3}
slice2 := slice // shares underlying array
slice2[0] = 99
fmt.Println(slice[0]) // 99 (both point to same array)
```

### 🔥 Q: Slice internals and append gotcha?

```go
// Slice = {pointer, len, cap}
s := make([]int, 3, 5) // len=3, cap=5
// Underlying array: [0 0 0 _ _]

s = append(s, 4) // len=4, cap=5 (no reallocation)
s = append(s, 5) // len=5, cap=5
s = append(s, 6) // len=6, cap=10 (reallocates, copies)
```

**Aliasing gotcha:**

```go
a := []int{1, 2, 3, 4, 5}
b := a[1:3] // [2 3]
b[0] = 99
fmt.Println(a) // [1 99 3 4 5] — b modifies a's underlying array

// After append (if capacity exceeded), b gets new array
b = append(b, 100)
b[0] = 77
fmt.Println(a) // [1 99 3 4 5] — no change (b now has separate array)
```

### 🔥 Q: Are maps safe for concurrent access?

**No.** Maps are NOT safe for concurrent reads + writes.

```go
// RACE
m := make(map[string]int)
go func() { m["a"] = 1 }()
go func() { fmt.Println(m["a"]) }()

// FIX 1: sync.Mutex
var mu sync.Mutex
go func() { mu.Lock(); m["a"] = 1; mu.Unlock() }()

// FIX 2: sync.Map (for concurrent read-heavy workloads)
var m sync.Map
m.Store("a", 1)
val, ok := m.Load("a")
```

### ⭐ Q: What happens when reading a non-existent map key?

```go
m := map[string]int{"a": 1}
val := m["b"]        // val = 0 (zero value)
val, ok := m["b"]   // val = 0, ok = false
```

---

## Interfaces & Type System

### ⭐ Q: Type assertion and type switch?

```go
var i interface{} = "hello"

// Type assertion
s := i.(string)       // panics if wrong type
s, ok := i.(string)   // safe: ok = false if wrong

// Type switch
switch v := i.(type) {
case int:
    fmt.Println("int:", v)
case string:
    fmt.Println("string:", v)
default:
    fmt.Println("unknown type")
}
```

### 💡 Q: Empty interface vs generic (Go 1.18+)?

```go
// Before Go 1.18
func printAny(v interface{}) {
    fmt.Println(v)
}

// Go 1.18+ generics
func printGeneric[T any](v T) {
    fmt.Println(v)
}

func max[T constraints.Ordered](a, b T) T {
    if a > b { return a }
    return b
}
```

---

## Error Handling

### 🔥 Q: Idiomatic error handling in Go?

```go
// Always wrap errors with context
file, err := os.Open("config.yaml")
if err != nil {
    return fmt.Errorf("open config: %w", err) // %w wraps error
}
defer file.Close()

// Check wrapped errors
if errors.Is(err, os.ErrNotExist) {
    // Handle missing file
}

// Extract specific error type
var perr *os.PathError
if errors.As(err, &perr) {
    fmt.Println("Failed path:", perr.Path)
}
```

### ⭐ Q: Sentinel errors vs custom error types?

```go
// Sentinel error
var ErrNotFound = errors.New("not found")

func Get(key string) (string, error) {
    if !exists(key) {
        return "", ErrNotFound
    }
    return value, nil
}

// Check: errors.Is(err, ErrNotFound)

// Custom error type
type ValidationError struct {
    Field string
    Issue string
}

func (e ValidationError) Error() string {
    return fmt.Sprintf("validation failed on %s: %s", e.Field, e.Issue)
}

// Check: var ve ValidationError; errors.As(err, &ve)
```

### ⭐ Q: Error wrapping chain?

```go
// Wrap errors to preserve context
func readConfig() error {
    data, err := os.ReadFile("config.yaml")
    if err != nil {
        return fmt.Errorf("read config: %w", err)
    }
    if err := yaml.Unmarshal(data, &cfg); err != nil {
        return fmt.Errorf("parse config: %w", err)
    }
    return nil
}

// Unwrap chain
err := readConfig()
// Error: "parse config: read config: open config.yaml: no such file or directory"

if errors.Is(err, os.ErrNotExist) {
    // Detects wrapped os.ErrNotExist
}
```

---

## Defer, Panic, Recover

### 🔥 Q: How does `defer` work?

```go
func example() {
    defer fmt.Println("third")
    defer fmt.Println("second")
    fmt.Println("first")
}
// Output: first, second, third (LIFO)

// Use for cleanup
func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil { return err }
    defer f.Close() // runs even if error occurs below

    // Process file...
    return nil
}
```

**Defer gotcha:**

```go
// BAD: defer in loop
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close() // defers accumulate, only run at end
}

// GOOD: wrap in function
for _, file := range files {
    func() {
        f, _ := os.Open(file)
        defer f.Close() // runs at end of each iteration
    }()
}
```

### ⭐ Q: Panic and recover?

```go
func recoverExample() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()
    panic("something went wrong")
}

// Only recover in top-level handlers (HTTP server, worker pool)
// Do NOT use for normal control flow
```

---

## HTTP Server & Client

### 🔥 Q: Production-ready HTTP server with graceful shutdown?

```go
func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", healthHandler)
    mux.HandleFunc("/api/v1/data", dataHandler)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      mux,
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    // Graceful shutdown
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("Server failed: %v", err)
        }
    }()

    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit

    log.Println("Shutting down server...")
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    log.Println("Server exited")
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(map[string]string{
        "status": "healthy",
        "time":   time.Now().Format(time.RFC3339),
    })
}
```

### ⭐ Q: HTTP client with context, retry, and exponential backoff?

```go
func getWithRetry(ctx context.Context, url string, maxRetries int) (*http.Response, error) {
    client := &http.Client{Timeout: 10 * time.Second}
    var lastErr error

    for attempt := 0; attempt <= maxRetries; attempt++ {
        req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
        if err != nil {
            return nil, err
        }

        resp, err := client.Do(req)
        if err == nil && resp.StatusCode < 500 {
            return resp, nil
        }
        if resp != nil {
            resp.Body.Close()
        }

        lastErr = err
        if attempt < maxRetries {
            backoff := time.Duration(1<<uint(attempt)) * time.Second
            select {
            case <-time.After(backoff):
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
    }
    return nil, fmt.Errorf("failed after %d retries: %w", maxRetries, lastErr)
}
```

### ⭐ Q: Middleware pattern for HTTP handlers?

```go
// Middleware signature
func middleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // Before handler
        next(w, r)
        // After handler
    }
}

// Logging middleware
func loggingMiddleware(next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        log.Printf("%s %s", r.Method, r.URL.Path)
        next(w, r)
        log.Printf("completed in %v", time.Since(start))
    }
}

// Chain middlewares
mux.HandleFunc("/api", loggingMiddleware(authMiddleware(dataHandler)))
```

---

## CLI Tools

### ⭐ Q: Building a CLI with Cobra (kubectl, helm, docker all use it)?

```go
package main

import (
    "fmt"
    "github.com/spf13/cobra"
    "os"
)

var (
    version   string
    namespace string
)

var rootCmd = &cobra.Command{
    Use:   "deployer",
    Short: "A deployment tool for Kubernetes",
}

var deployCmd = &cobra.Command{
    Use:   "deploy",
    Short: "Deploy application",
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Deploying version %s to namespace %s\n", version, namespace)
        // kubectl apply logic here
        return nil
    },
}

func init() {
    deployCmd.Flags().StringVarP(&version, "version", "v", "", "App version (required)")
    deployCmd.Flags().StringVarP(&namespace, "namespace", "n", "default", "K8s namespace")
    deployCmd.MarkFlagRequired("version")
    rootCmd.AddCommand(deployCmd)
}

func main() {
    if err := rootCmd.Execute(); err != nil {
        os.Exit(1)
    }
}
```

### 💡 Q: Alternative: stdlib `flag` package?

```go
import "flag"

var (
    host = flag.String("host", "localhost", "Server host")
    port = flag.Int("port", 8080, "Server port")
)

func main() {
    flag.Parse()
    fmt.Printf("Starting server on %s:%d\n", *host, *port)
}
// Run: ./app -host=0.0.0.0 -port=9090
```

### ⭐ Q: Running external commands (kubectl, docker)?

```go
import "os/exec"

func runCommand(name string, args ...string) error {
    cmd := exec.Command(name, args...)
    cmd.Stdout = os.Stdout
    cmd.Stderr = os.Stderr
    return cmd.Run()
}

// Example
err := runCommand("kubectl", "apply", "-f", "deployment.yaml")
if err != nil {
    log.Fatal(err)
}

// Capture output
output, err := exec.Command("kubectl", "get", "pods", "-o", "json").Output()
```

---

## Testing & Benchmarking

### 🔥 Q: Table-driven tests (Go convention)?

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -2, -3},
        {"zero", 0, 5, 5},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("add(%d, %d) = %d; want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

### ⭐ Q: Mocking HTTP requests (httptest)?

```go
import "net/http/httptest"

func TestHealthHandler(t *testing.T) {
    req := httptest.NewRequest("GET", "/health", nil)
    rr := httptest.NewRecorder()
    
    healthHandler(rr, req)
    
    if status := rr.Code; status != http.StatusOK {
        t.Errorf("handler returned wrong status code: got %v want %v", status, http.StatusOK)
    }
    
    expected := `{"status":"healthy"}`
    if !strings.Contains(rr.Body.String(), "healthy") {
        t.Errorf("handler returned unexpected body: got %v", rr.Body.String())
    }
}
```

### ⭐ Q: Benchmarking?

```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        add(10, 20)
    }
}

// Run: go test -bench=. -benchmem
// Output:
// BenchmarkAdd-8   1000000000   0.25 ns/op   0 B/op   0 allocs/op
```

### 💡 Q: Using testify for assertions?

```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestDivide(t *testing.T) {
    result, err := divide(10, 2)
    require.NoError(t, err) // stops test if error
    assert.Equal(t, 5, result)
    
    _, err = divide(10, 0)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "zero")
}
```

### 💡 Q: Integration tests with test containers?

```go
// Use testcontainers-go for real DB/Redis in tests
func TestWithPostgres(t *testing.T) {
    ctx := context.Background()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:15",
        ExposedPorts: []string{"5432/tcp"},
        Env: map[string]string{
            "POSTGRES_PASSWORD": "test",
        },
    }
    postgres, _ := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: req,
        Started:          true,
    })
    defer postgres.Terminate(ctx)
    
    // Run tests against real postgres...
}
```

---

## Modules & Dependencies

### 🔥 Q: Go modules basics?

```bash
go mod init github.com/user/repo  # Create go.mod
go get github.com/gin-gonic/gin   # Add dependency
go mod tidy                        # Remove unused deps
go mod vendor                      # Copy deps to vendor/
```

**go.mod:**

```go
module github.com/user/myapp

go 1.22

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/spf13/cobra v1.8.0
)
```

### 💡 Q: Semantic import versioning (v2+)?

```go
// v0/v1: import "github.com/pkg/foo"
// v2+:   import "github.com/pkg/foo/v2"

go get github.com/pkg/foo/v2@v2.0.0
```

### 💡 Q: Private modules?

```bash
# Set GOPRIVATE to skip proxy/checksum for private repos
export GOPRIVATE=github.com/mycompany/*
go get github.com/mycompany/private-repo
```

---

## Building for Containers

### 🔥 Q: Cross-compilation?

```bash
# Build for Linux AMD64 from macOS
GOOS=linux GOARCH=amd64 go build -o myapp-linux main.go

# Static binary (no libc dependency)
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o myapp .
```

### 🔥 Q: Minimal Docker image (distroless/scratch)?

```dockerfile
# Multi-stage build
FROM golang:1.22 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -ldflags="-s -w" -o myapp .

# Final image (distroless)
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/myapp /
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/myapp"]

# OR use scratch (even smaller, no shell)
FROM scratch
COPY --from=builder /app/myapp /myapp
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
EXPOSE 8080
ENTRYPOINT ["/myapp"]
```

### 💡 Q: Embedding files (Go 1.16+)?

```go
import "embed"

//go:embed config.yaml templates/*
var content embed.FS

func main() {
    data, _ := content.ReadFile("config.yaml")
    fmt.Println(string(data))
}
```

---

## Common Gotchas

### 🔥 Q: Loop variable capture (pre-Go 1.22)?

```go
// Before Go 1.22 — all goroutines print "3"
for i := 0; i < 3; i++ {
    go func() { fmt.Println(i) }()
}

// Fix: pass as argument
for i := 0; i < 3; i++ {
    go func(n int) { fmt.Println(n) }(i)
}

// Go 1.22+: loop variables are now per-iteration (gotcha fixed!)
```

### ⭐ Q: Nil interface != nil value?

```go
func returnsNil() error {
    var p *MyError = nil
    return p // interface{type: *MyError, value: nil}
}

err := returnsNil()
fmt.Println(err == nil) // false! (interface has type)

// Fix: return explicit nil
func returnsNil() error {
    var p *MyError = nil
    if p == nil { return nil }
    return p
}
```

### ⭐ Q: Goroutine leaks?

```go
// LEAK: goroutine blocks forever
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // blocks forever, no sender
        fmt.Println(val)
    }()
}

// Fix: use context for cancellation
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return
        }
    }()
}
```

### 💡 Q: Slice reallocation after append?

```go
a := make([]int, 0, 3)
b := append(a, 1, 2, 3) // len=3, cap=3
c := append(b, 4)       // len=4, cap=6 (reallocated)

b[0] = 99
fmt.Println(c[0]) // 1 (c has different backing array)
```

---

## Coding Interview Questions

### ⭐ Q: Implement a rate limiter (token bucket).

```go
type RateLimiter struct {
    tokens    int
    maxTokens int
    mu        sync.Mutex
    ticker    *time.Ticker
}

func NewRateLimiter(rate int, per time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens:    rate,
        maxTokens: rate,
        ticker:    time.NewTicker(per / time.Duration(rate)),
    }
    go rl.refill()
    return rl
}

func (rl *RateLimiter) refill() {
    for range rl.ticker.C {
        rl.mu.Lock()
        if rl.tokens < rl.maxTokens {
            rl.tokens++
        }
        rl.mu.Unlock()
    }
}

func (rl *RateLimiter) Allow() bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    if rl.tokens > 0 {
        rl.tokens--
        return true
    }
    return false
}

func main() {
    limiter := NewRateLimiter(5, time.Second) // 5 requests/sec
    for i := 0; i < 10; i++ {
        if limiter.Allow() {
            fmt.Println("Request", i, "allowed")
        } else {
            fmt.Println("Request", i, "rate limited")
        }
        time.Sleep(100 * time.Millisecond)
    }
}
```

### ⭐ Q: Implement a concurrent worker pool that processes jobs.

```go
type Job struct {
    ID   int
    Data string
}

type Result struct {
    Job   Job
    Output string
}

func worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job.ID)
        time.Sleep(time.Second) // simulate work
        results <- Result{Job: job, Output: fmt.Sprintf("processed-%s", job.Data)}
    }
}

func main() {
    numWorkers := 3
    jobs := make(chan Job, 10)
    results := make(chan Result, 10)

    // Start workers
    for w := 1; w <= numWorkers; w++ {
        go worker(w, jobs, results)
    }

    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("task-%d", j)}
    }
    close(jobs)

    // Collect results
    for r := 1; r <= 5; r++ {
        res := <-results
        fmt.Printf("Result: Job %d -> %s\n", res.Job.ID, res.Output)
    }
}
```

### 💡 Q: Read a large file concurrently (split into chunks).

```go
func readFileConcurrent(filename string, chunkSize int64) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()

    stat, _ := file.Stat()
    fileSize := stat.Size()
    numChunks := int((fileSize + chunkSize - 1) / chunkSize)

    var wg sync.WaitGroup
    for i := 0; i < numChunks; i++ {
        wg.Add(1)
        go func(chunkIndex int) {
            defer wg.Done()
            offset := int64(chunkIndex) * chunkSize
            buffer := make([]byte, chunkSize)
            
            f, _ := os.Open(filename)
            defer f.Close()
            f.Seek(offset, 0)
            n, _ := f.Read(buffer)
            
            fmt.Printf("Chunk %d: read %d bytes\n", chunkIndex, n)
            // Process buffer...
        }(i)
    }
    wg.Wait()
    return nil
}
```

### 🔥 Q: Detect and fix a data race in this code.

```go
// RACE CONDITION
func buggyCounter() {
    var count int
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            count++ // RACE: multiple goroutines writing
        }()
    }
    wg.Wait()
    fmt.Println("Count:", count) // Will be < 1000
}

// FIX 1: Mutex
func fixedCounterMutex() {
    var count int
    var mu sync.Mutex
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()
            count++
            mu.Unlock()
        }()
    }
    wg.Wait()
    fmt.Println("Count:", count) // 1000
}

// FIX 2: Atomic
func fixedCounterAtomic() {
    var count int64
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&count, 1)
        }()
    }
    wg.Wait()
    fmt.Println("Count:", count) // 1000
}
```

---

## Key Resources

### Official Documentation
- **A Tour of Go** — https://go.dev/tour (interactive tutorial)
- **Effective Go** — https://go.dev/doc/effective_go (idioms and conventions)
- **Go Memory Model** — https://go.dev/ref/mem (concurrency guarantees)
- **Go Spec** — https://go.dev/ref/spec (language reference)

### Learning Resources
- **Go by Example** — https://gobyexample.com (code snippets)
- **Learn Go with Tests** — https://quii.gitbook.io/learn-go-with-tests (TDD approach)
- **Concurrency in Go (book)** — Katherine Cox-Buday (concurrency patterns)
- **The Go Programming Language (book)** — Donovan & Kernighan

### DevOps-Specific
- **Programming Kubernetes (book)** — For writing operators and controllers
- **client-go** — https://github.com/kubernetes/client-go (K8s Go client)
- **Cobra** — https://github.com/spf13/cobra (CLI framework)
- **Prometheus client_golang** — https://github.com/prometheus/client_golang

### Tools & Commands
```bash
go run -race main.go              # Run with race detector
go test -bench=. -benchmem        # Benchmark tests
go tool pprof                     # CPU/memory profiling
go build -gcflags="-m"            # Escape analysis
GODEBUG=gctrace=1 ./app           # GC trace
go mod graph                      # Dependency graph
```
