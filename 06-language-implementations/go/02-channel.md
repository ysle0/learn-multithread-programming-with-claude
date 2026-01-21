# Channels in Go

Channels are Go's way of allowing goroutines to communicate and synchronize. They're type-safe conduits for passing values between goroutines.

## Basic Concepts

### Creating Channels

```go
ch := make(chan int)           // Unbuffered
ch := make(chan int, 5)        // Buffered (capacity 5)
ch := make(chan string, 100)   // Buffered string channel
```

### Sending and Receiving

```go
ch := make(chan int)

// Send
go func() {
    ch <- 42  // Send 42 to channel
}()

// Receive
value := <-ch  // Receive from channel

// Receive and discard
<-ch
```

## Unbuffered Channels

### Synchronization

```go
func main() {
    ch := make(chan string)
    
    go func() {
        ch <- "hello"  // Blocks until received
    }()
    
    msg := <-ch  // Blocks until sent
    fmt.Println(msg)
}
```

## Buffered Channels

### Non-blocking Sends (Until Full)

```go
ch := make(chan int, 3)

ch <- 1  // Doesn't block
ch <- 2  // Doesn't block
ch <- 3  // Doesn't block
// ch <- 4  // Would block (buffer full)

fmt.Println(<-ch)  // 1
fmt.Println(<-ch)  // 2
```

## Closing Channels

### Sender Closes

```go
func sender(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)  // Sender closes
}

func main() {
    ch := make(chan int)
    go sender(ch)
    
    // Receive until closed
    for val := range ch {
        fmt.Println(val)
    }
}
```

### Checking if Closed

```go
val, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}
```

## Direction (Send-only, Receive-only)

```go
// Send-only channel
func sender(ch chan<- int) {
    ch <- 42
    // val := <-ch  // Compile error
}

// Receive-only channel
func receiver(ch <-chan int) {
    val := <-ch
    // ch <- 42  // Compile error
}
```

## Common Patterns

### Worker Pool

```go
func worker(jobs <-chan int, results chan<- int) {
    for job := range jobs {
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 100)
    results := make(chan int, 100)
    
    // Start workers
    for w := 0; w < 3; w++ {
        go worker(jobs, results)
    }
    
    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Receive results
    for r := 1; r <= 5; r++ {
        fmt.Println(<-results)
    }
}
```

### Pipeline

```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
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

func main() {
    for n := range square(generator(2, 3, 4)) {
        fmt.Println(n)  // 4, 9, 16
    }
}
```

## Navigation

- [Back to Go Overview](./README.md)
- Previous: [Goroutines](./01-goroutine.md)
- Next: [Select Statement](./03-select.md)
