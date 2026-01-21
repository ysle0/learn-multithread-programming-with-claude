# Sync Package in Go

The `sync` package provides traditional synchronization primitives for when channels aren't the right tool.

## Mutex

### Basic Mutex

```go
import "sync"

type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    c.value++
    c.mu.Unlock()
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

## RWMutex (Reader-Writer Lock)

```go
type Cache struct {
    mu    sync.RWMutex
    data  map[string]string
}

func (c *Cache) Get(key string) string {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.data[key]
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

## WaitGroup

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        fmt.Printf("Worker %d\n", id)
    }(i)
}

wg.Wait()
```

## Once (Run Once)

```go
var once sync.Once
var instance *Singleton

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

## sync.Map (Concurrent Map)

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
if val, ok := m.Load("key"); ok {
    fmt.Println(val)
}

// LoadOrStore
actual, loaded := m.LoadOrStore("key", "value")

// Delete
m.Delete("key")

// Range
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true  // continue iteration
})
```

## Navigation

- [Back to Go Overview](./README.md)
- Previous: [Select Statement](./03-select.md)
- Next: [Context Package](./05-context.md)
