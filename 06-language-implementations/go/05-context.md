# Context Package in Go

The `context` package provides a way to pass cancellation signals, deadlines, and request-scoped values across API boundaries.

## Basic Context

### Background and TODO

```go
import "context"

// Root context
ctx := context.Background()

// When you don't know which context to use
ctx := context.TODO()
```

## Cancellation

### WithCancel

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    go func(ctx context.Context) {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Cancelled")
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }(ctx)
    
    time.Sleep(2 * time.Second)
    cancel()  // Cancel goroutine
    time.Sleep(time.Second)
}
```

## Timeouts

### WithTimeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("Work done")
case <-ctx.Done():
    fmt.Println("Timeout:", ctx.Err())
}
```

### WithDeadline

```go
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("Work done")
case <-ctx.Done():
    fmt.Println("Deadline exceeded")
}
```

## Context Values

### WithValue

```go
type key string

ctx := context.WithValue(context.Background(), key("user"), "alice")

// Retrieve value
if user, ok := ctx.Value(key("user")).(string); ok {
    fmt.Println("User:", user)
}
```

## Best Practices

### Function Signature

```go
// GOOD: Context as first parameter
func DoWork(ctx context.Context, arg string) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // Work
        return nil
    }
}
```

### Always Respect Context

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return  // Exit when cancelled
        default:
            // Do work
        }
    }
}
```

## Complete Example: HTTP Request with Timeout

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "time"
)

func fetchWithTimeout(url string, timeout time.Duration) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    
    body, err := io.ReadAll(resp.Body)
    if err != nil {
        return "", err
    }
    
    return string(body), nil
}

func main() {
    body, err := fetchWithTimeout("https://google.com", 5*time.Second)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Printf("Fetched %d bytes\n", len(body))
}
```

## Navigation

- [Back to Go Overview](./README.md)
- Previous: [Sync Package](./04-sync-package.md)
- [Back to Language Implementations](../)
