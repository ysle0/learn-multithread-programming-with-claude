# Go Concurrency

Go was designed from the ground up with concurrency as a first-class citizen. Its lightweight goroutines and channels make concurrent programming natural and efficient.

## Overview

Go's concurrency model is based on CSP (Communicating Sequential Processes):
- **Goroutines**: Lightweight threads managed by Go runtime
- **Channels**: Type-safe communication between goroutines
- **Select**: Multiplexing channel operations
- **Sync Package**: Traditional synchronization primitives
- **Context**: Cancellation and deadlines

## Core Components

### 1. [Goroutines](./01-goroutine.md)
- Creating goroutines with `go` keyword
- Lightweight concurrency
- Runtime scheduler (M:N model)
- Goroutine lifecycle

### 2. [Channels](./02-channel.md)
- Unbuffered and buffered channels
- Sending and receiving
- Closing channels
- Range over channels

### 3. [Select Statement](./03-select.md)
- Multiplexing channel operations
- Non-blocking operations
- Timeouts and defaults
- Select patterns

### 4. [Sync Package](./04-sync-package.md)
- `sync.Mutex` and `sync.RWMutex`
- `sync.WaitGroup`
- `sync.Once`
- `sync.Pool`

### 5. [Context Package](./05-context.md)
- Cancellation propagation
- Timeouts and deadlines
- Request-scoped values
- Context best practices

## Quick Comparison with Other Languages

| Feature | Go | Comparison |
|---------|-----|------------|
| **Concurrency Model** | Goroutines + Channels (CSP) | Most natural for concurrent programming |
| **Thread Weight** | ~2KB per goroutine | Lightest - can have millions |
| **Communication** | Channels (built-in) | Most elegant message passing |
| **Syntax** | `go` keyword | Simplest to create concurrent tasks |
| **Learning Curve** | Gentle | Easier than C++, simpler than async/await |

## Key Principles

### 1. Goroutines are Cheap

```go
// Can easily create millions of goroutines
for i := 0; i < 1000000; i++ {
    go func(n int) {
        // Work
    }(i)
}
```

### 2. Share Memory by Communicating

```go
// GOOD: Use channels to communicate
ch := make(chan int)
go func() {
    ch <- 42  // Send
}()
result := <-ch  // Receive

// Less preferred: Shared memory with mutex
var mu sync.Mutex
var shared int
mu.Lock()
shared = 42
mu.Unlock()
```

### 3. Don't Communicate by Sharing Memory

Go's philosophy: "Don't communicate by sharing memory; share memory by communicating."

## Common Patterns

### Worker Pool

```go
func workerPool(jobs <-chan int, results chan<- int) {
    for w := 0; w < 3; w++ {
        go func() {
            for job := range jobs {
                results <- job * 2
            }
        }()
    }
}
```

### Pipeline

```go
func generator() <-chan int {
    ch := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            ch <- i
        }
        close(ch)
    }()
    return ch
}

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

// Usage
for result := range square(generator()) {
    fmt.Println(result)
}
```

## Best Practices

1. **Always handle goroutine completion**: Use WaitGroups or channels
2. **Close channels from sender side**: Receivers should never close
3. **Use select for timeouts**: Don't block indefinitely
4. **Pass context for cancellation**: First parameter of functions
5. **Check for closed channels**: Test receive operations
6. **Avoid goroutine leaks**: Ensure all goroutines can exit
7. **Use buffered channels wisely**: For known capacity

## Common Pitfalls

### 1. Goroutine Leaks

```go
// BAD: Goroutine never exits
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // Blocks forever if nothing sent
    }()
}

// GOOD: With timeout
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-time.After(time.Second):
            return
        }
    }()
}
```

### 2. Closing Channel from Receiver

```go
// BAD: Receiver closes
go func() {
    for val := range ch {
        process(val)
    }
    close(ch)  // WRONG!
}()

// GOOD: Sender closes
go func() {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)  // Correct
}()
```

### 3. Loop Variable Capture

```go
// BAD: Captures loop variable
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // May print wrong value
    }()
}

// GOOD: Pass as parameter
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

### 4. Sending on Closed Channel

```go
// BAD: Panic
ch := make(chan int)
close(ch)
ch <- 1  // Panic!

// GOOD: Check before closing
var once sync.Once
once.Do(func() {
    close(ch)
})
```

### 5. Race Conditions

```go
// BAD: Data race
var counter int
for i := 0; i < 1000; i++ {
    go func() {
        counter++  // Race!
    }()
}

// GOOD: Use atomic or mutex
var counter int64
for i := 0; i < 1000; i++ {
    go func() {
        atomic.AddInt64(&counter, 1)
    }()
}
```

## Performance Characteristics

### Goroutine Overhead
- **Creation**: ~1 microsecond
- **Memory**: ~2KB initial stack (growable)
- **Context switch**: Very fast (in-process)
- **Scalability**: Millions of goroutines possible

### Channel Performance
- **Unbuffered**: Synchronization point, slower
- **Buffered**: Faster, decouples sender/receiver
- **Send/receive**: ~100 nanoseconds

## Race Detection

```bash
# Run with race detector
go run -race main.go

# Build with race detector
go build -race

# Test with race detector
go test -race
```

## Tools and Debugging

### pprof for Profiling

```go
import _ "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // Your program
}

// Access at http://localhost:6060/debug/pprof/
```

### Goroutine Profiling

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### Tracing

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// Your code
```

## Modern Go Features

### Generics (Go 1.18+)

```go
func SendSlice[T any](ch chan<- T, slice []T) {
    for _, v := range slice {
        ch <- v
    }
    close(ch)
}
```

### Error Handling with Goroutines

```go
type Result struct {
    Value int
    Error error
}

func compute() Result {
    ch := make(chan Result, 1)
    go func() {
        // Do work
        ch <- Result{Value: 42}
    }()
    return <-ch
}
```

## Further Reading

- **Go Concurrency Patterns** by Rob Pike
- **Concurrency in Go** by Katherine Cox-Buday
- Go Blog: https://go.dev/blog/
- Effective Go: https://go.dev/doc/effective_go

## Navigation

- [Back to Language Implementations](../)
- Next Topics:
  - [Goroutines](./01-goroutine.md)
  - [Channels](./02-channel.md)
  - [Select Statement](./03-select.md)
  - [Sync Package](./04-sync-package.md)
  - [Context Package](./05-context.md)
