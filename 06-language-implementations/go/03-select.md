# Select Statement in Go

The `select` statement lets a goroutine wait on multiple channel operations, choosing whichever is ready first.

## Basic Select

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        time.Sleep(time.Second)
        ch1 <- "from ch1"
    }()
    
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "from ch2"
    }()
    
    select {
    case msg1 := <-ch1:
        fmt.Println(msg1)
    case msg2 := <-ch2:
        fmt.Println(msg2)
    }
}
```

## Default Case (Non-blocking)

```go
select {
case msg := <-ch:
    fmt.Println(msg)
default:
    fmt.Println("No message")
}
```

## Timeout Pattern

```go
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(time.Second):
    fmt.Println("Timeout!")
}
```

## Complete Example

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    c1 := make(chan string)
    c2 := make(chan string)
    
    go func() {
        for {
            select {
            case msg1 := <-c1:
                fmt.Println("Received:", msg1)
            case msg2 := <-c2:
                fmt.Println("Received:", msg2)
            case <-time.After(2 * time.Second):
                fmt.Println("Timeout")
                return
            }
        }
    }()
    
    c1 <- "Hello"
    time.Sleep(time.Second)
    c2 <- "World"
    time.Sleep(3 * time.Second)
}
```

## Navigation

- [Back to Go Overview](./README.md)
- Previous: [Channels](./02-channel.md)
- Next: [Sync Package](./04-sync-package.md)
