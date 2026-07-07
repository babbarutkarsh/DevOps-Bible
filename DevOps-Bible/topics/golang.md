---
title: Go (Golang)
nav_order: 70
description: "Go (Golang) — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Go (Golang) for DevOps — Interview Preparation

## Why Go for DevOps

| Feature | Why It Matters |
|---------|---------------|
| **Static binary** | No runtime deps — perfect for containers |
| **Fast compilation** | Quick CI builds |
| **Concurrency** | Goroutines for parallel operations |
| **Strong stdlib** | HTTP, JSON, crypto built-in |
| **Cross-compilation** | `GOOS=linux GOARCH=amd64 go build` |

**Tools in Go:** Kubernetes, Docker, Terraform, Prometheus, Helm, Istio, Vault, ArgoCD, etcd, Traefik, Cilium

---

## Go Fundamentals

```go
// Structs, methods, interfaces
type Server struct {
    Name string `json:"name"`
    IP   string `json:"ip"`
    Port int    `json:"port"`
}

func (s *Server) Address() string {
    return fmt.Sprintf("%s:%d", s.IP, s.Port)
}

// Interfaces — implicit implementation
type HealthChecker interface { CheckHealth() error }
```

---

## Concurrency

```go
// Goroutines + WaitGroup + Channels
func checkServers(urls []string) {
    var wg sync.WaitGroup
    results := make(chan string, len(urls))
    for _, url := range urls {
        wg.Add(1)
        go func(u string) {
            defer wg.Done()
            resp, err := http.Get(u)
            if err != nil { results <- fmt.Sprintf("%s: DOWN", u); return }
            defer resp.Body.Close()
            results <- fmt.Sprintf("%s: %d", u, resp.StatusCode)
        }(url)
    }
    go func() { wg.Wait(); close(results) }()
    for r := range results { fmt.Println(r) }
}

// Context for timeouts
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
defer cancel()
```

---

## Error Handling

```go
// Always wrap errors with context
file, err := os.Open("config.yaml")
if err != nil {
    return fmt.Errorf("open config: %w", err)
}
defer file.Close()

// Check wrapped errors
if errors.Is(err, os.ErrNotExist) { /* handle */ }
```

---

## HTTP Server & Client

```go
// Health endpoint
func healthHandler(w http.ResponseWriter, r *http.Request) {
    json.NewEncoder(w).Encode(map[string]string{
        "status": "healthy", "time": time.Now().Format(time.RFC3339),
    })
}

server := &http.Server{
    Addr: ":8080", ReadTimeout: 5 * time.Second, WriteTimeout: 10 * time.Second,
}

// Client with retry
func getWithRetry(url string, retries int) (*http.Response, error) {
    client := &http.Client{Timeout: 10 * time.Second}
    for i := 0; i < retries; i++ {
        resp, err := client.Get(url)
        if err == nil && resp.StatusCode < 500 { return resp, nil }
        if resp != nil { resp.Body.Close() }
        time.Sleep(time.Duration(1<<uint(i)) * time.Second)
    }
    return nil, fmt.Errorf("failed after %d retries", retries)
}
```

---

## CLI Tools (Cobra — used by kubectl, helm, docker)

```go
var rootCmd = &cobra.Command{Use: "deployer", Short: "Deployment CLI"}
var deployCmd = &cobra.Command{
    Use: "deploy", Short: "Deploy app",
    RunE: func(cmd *cobra.Command, args []string) error {
        fmt.Printf("Deploying %s to %s\n", version, namespace)
        return nil
    },
}
func init() {
    deployCmd.Flags().StringVarP(&version, "version", "v", "", "Version")
    rootCmd.AddCommand(deployCmd)
}
```

---

## Testing

```go
func TestServer_Address(t *testing.T) {
    s := Server{IP: "10.0.1.10", Port: 8080}
    got := s.Address()
    want := "10.0.1.10:8080"
    if got != want {
        t.Errorf("Address() = %q, want %q", got, want)
    }
}

// Table-driven tests (Go convention)
func TestHealthCheck(t *testing.T) {
    tests := []struct{name, url string; want bool}{
        {"healthy", "http://ok.test", true},
        {"down", "http://down.test", false},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if got := checkHealth(tt.url); got != tt.want {
                t.Errorf("got %v, want %v", got, tt.want)
            }
        })
    }
}
// Run: go test ./... -v -cover
```

---

## Key Resources

- **A Tour of Go** — https://go.dev/tour
- **Go by Example** — https://gobyexample.com
- **Effective Go** — https://go.dev/doc/effective_go
- **Learn Go with Tests** — https://quii.gitbook.io/learn-go-with-tests
- **Programming Kubernetes (book)** — For writing operators
- **client-go** — https://github.com/kubernetes/client-go
