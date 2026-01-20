# Goroutines in Go

Goroutines are lightweight threads managed by the Go runtime. They're the foundation of Go's concurrency model.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [Creating Goroutines](#creating-goroutines)
- [Goroutine Lifecycle](#goroutine-lifecycle)
- [WaitGroups](#waitgroups)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is a Goroutine?

```go
package main

import (
    "fmt"
    "time"
)

func hello() {
    fmt.Println("Hello from goroutine!")
}

func main() {
    go hello()  // Launch goroutine
    
    time.Sleep(time.Second)  // Wait (not ideal, see WaitGroup)
    fmt.Println("Main function")
}
```

### How Goroutines Work

- **M:N Scheduler**: Many goroutines on fewer OS threads
- **Initial Stack**: ~2KB (grows/shrinks automatically)
- **Cooperative**: Goroutines yield at function calls, channel ops, etc.
- **Cheap**: Can create millions

## Creating Goroutines

### Anonymous Functions

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // Launch with anonymous function
    go func() {
        fmt.Println("Anonymous goroutine")
    }()
    
    // With parameters
    message := "Hello"
    go func(msg string) {
        fmt.Println(msg)
    }(message)
    
    time.Sleep(time.Second)
}
```

### Named Functions

```go
func printNumbers() {
    for i := 0; i < 5; i++ {
        fmt.Println(i)
    }
}

func main() {
    go printNumbers()
    time.Sleep(time.Second)
}
```

### Methods as Goroutines

```go
type Worker struct {
    ID int
}

func (w Worker) Work() {
    fmt.Printf("Worker %d working\n", w.ID)
}

func main() {
    w := Worker{ID: 1}
    go w.Work()
    time.Sleep(time.Millisecond * 100)
}
```

## Goroutine Lifecycle

### Simple Goroutine

```go
func main() {
    done := make(chan bool)
    
    go func() {
        fmt.Println("Goroutine running")
        time.Sleep(time.Second)
        fmt.Println("Goroutine done")
        done <- true
    }()
    
    <-done  // Wait for goroutine
    fmt.Println("Main done")
}
```

## WaitGroups

### Basic WaitGroup

```go
import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()  // Decrement counter when done
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)  // Increment counter
        go worker(i, &wg)
    }
    
    wg.Wait()  // Block until counter reaches 0
    fmt.Println("All workers done")
}
```

### WaitGroup with Error Handling

```go
func worker(id int, wg *sync.WaitGroup, errCh chan<- error) {
    defer wg.Done()
    
    if id == 3 {
        errCh <- fmt.Errorf("worker %d failed", id)
        return
    }
    
    fmt.Printf("Worker %d completed\n", id)
}

func main() {
    var wg sync.WaitGroup
    errCh := make(chan error, 5)
    
    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(i, &wg, errCh)
    }
    
    wg.Wait()
    close(errCh)
    
    for err := range errCh {
        fmt.Println("Error:", err)
    }
}
```

## Comparison with Other Languages

### Go vs. C++
```go
// Go: Very lightweight
go doWork()

// C++ equivalent:
// std::thread t(doWork);
// t.join();
// (Much heavier - OS thread)
```

### Go vs. C#
```go
// Go: Goroutine
go doWork()

// C# equivalent:
// Task.Run(() => doWork());
// (Task uses thread pool, still heavier)
```

## Best Practices

### 1. Always Coordinate Goroutine Completion

```go
// GOOD: Using WaitGroup
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
wg.Wait()

// GOOD: Using channel
done := make(chan bool)
go func() {
    work()
    done <- true
}()
<-done
```

### 2. Don't Sleep to Wait

```go
// BAD: Sleeping to wait
go work()
time.Sleep(time.Second)

// GOOD: Use synchronization
done := make(chan bool)
go func() {
    work()
    done <- true
}()
<-done
```

### 3. Pass Data via Parameters

```go
// GOOD: Pass as parameter
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}

// BAD: Capture loop variable
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // May print wrong value
    }()
}
```

## Common Pitfalls

### Loop Variable Capture

```go
// BAD: All goroutines see same variable
for _, val := range values {
    go func() {
        fmt.Println(val)  // Wrong!
    }()
}

// GOOD: Pass as parameter
for _, val := range values {
    go func(v string) {
        fmt.Println(v)
    }(val)
}
```

### Goroutine Leaks

```go
// BAD: Goroutine leaks if channel never receives
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // Blocks forever
    }()
}

// GOOD: With timeout
func noLeak() {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-time.After(time.Second):
        }
    }()
}
```

## Complete Example: Concurrent Web Scraper

```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "sync"
)

func fetch(url string, wg *sync.WaitGroup, results chan<- string) {
    defer wg.Done()
    
    resp, err := http.Get(url)
    if err != nil {
        results <- fmt.Sprintf("%s: error - %v", url, err)
        return
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    results <- fmt.Sprintf("%s: %d bytes", url, len(body))
}

func main() {
    urls := []string{
        "https://golang.org",
        "https://google.com",
        "https://github.com",
    }
    
    var wg sync.WaitGroup
    results := make(chan string, len(urls))
    
    for _, url := range urls {
        wg.Add(1)
        go fetch(url, &wg, results)
    }
    
    wg.Wait()
    close(results)
    
    for result := range results {
        fmt.Println(result)
    }
}
```

## Navigation

- [Back to Go Overview](./README.md)
- Next: [Channels](./02-channel.md)
