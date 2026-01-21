# xsync - Go 동시성 프리미티브

## 📌 프로젝트 개요

**xsync**는 Go의 표준 `sync` 패키지를 확장하는 고성능 동시성 프리미티브 라이브러리입니다.

### 기본 정보
- **저장소**: https://github.com/puzpuzpuz/xsync
- **언어**: Go 1.18+ (Generics 활용)
- **라이선스**: Apache License 2.0
- **주요 개발자**: Andrey Pechkurov
- **활성 상태**: 활발히 유지보수 중
- **난이도**: ⭐⭐⭐

### 주요 특징
- **제네릭 기반**: 타입 안전한 자료구조
- **고성능**: 표준 라이브러리보다 빠른 성능
- **간단한 API**: 사용하기 쉬운 인터페이스
- **Lock-Free 알고리즘**: 확장성 우수
- **제로 의존성**: 순수 Go 구현

---

## 🎯 왜 xsync를 분석해야 하는가?

### 학습 가치

1. **Go 제네릭의 실전 활용**
   - 타입 안전한 동시성 자료구조
   - 성능과 안전성 모두 확보

2. **표준 라이브러리의 한계 극복**
   - `sync.Map`의 성능 문제 해결
   - 특화된 자료구조 제공

3. **Lock-Free 구현 학습**
   - Go에서의 Lock-Free 패턴
   - Atomic 연산 활용

4. **실무 적용 가능**
   - 프로덕션 레벨 품질
   - 다양한 사용 사례

---

## 🏗️ 아키텍처

### 주요 컴포넌트

```
xsync/
├── map.go              # MapOf - 제네릭 concurrent map
├── mpmcqueue.go        # MPMCQueue - Lock-free queue
├── rbmutex.go          # RBMutex - Range-based mutex
└── counter.go          # Counter - 고성능 카운터
```

---

## 🔬 핵심 컴포넌트 분석

### 1. MapOf - 제네릭 Concurrent Map

#### 개요
Go 1.18의 제네릭을 활용한 타입 안전한 concurrent map입니다.

#### 기본 사용법

```go
package main

import (
    "fmt"
    "github.com/puzpuzpuz/xsync/v3"
)

func main() {
    // 타입 안전한 맵 생성
    m := xsync.NewMapOf[string, int]()

    // 삽입
    m.Store("key1", 42)
    m.Store("key2", 100)

    // 조회
    val, ok := m.Load("key1")
    if ok {
        fmt.Println("key1:", val)  // key1: 42
    }

    // 삭제
    m.Delete("key2")

    // 반복
    m.Range(func(key string, value int) bool {
        fmt.Println(key, ":", value)
        return true  // continue
    })
}
```

#### 고급 기능

```go
// LoadOrStore - 원자적으로 조회 또는 삽입
actual, loaded := m.LoadOrStore("key", 42)
if loaded {
    fmt.Println("Key existed:", actual)
} else {
    fmt.Println("Key inserted:", actual)
}

// LoadAndDelete - 원자적으로 조회 및 삭제
val, loaded := m.LoadAndDelete("key")

// LoadOrCompute - 없으면 함수로 계산하여 삽입
val, loaded := m.LoadOrCompute("key", func() int {
    return expensiveComputation()
})

// Compute - 기존 값 기반으로 계산
m.Compute("counter", func(oldValue int, loaded bool) (newValue int, delete bool) {
    if !loaded {
        return 1, false  // 새 값 1
    }
    return oldValue + 1, false  // 기존 값 + 1
})

// Size - 대략적인 크기 (정확하지 않을 수 있음)
size := m.Size()
```

#### 내부 구조

```go
// 간소화된 버전
type MapOf[K comparable, V any] struct {
    // 여러 샤드로 분할 (lock contention 감소)
    shards    []*mapShard[K, V]
    shardMask uint64
    hasher    func(K) uint64
}

type mapShard[K comparable, V any] struct {
    mu    sync.RWMutex
    data  map[K]V
}

func NewMapOf[K comparable, V any]() *MapOf[K, V] {
    shardCount := runtime.NumCPU() * 4  // CPU 개수 기반
    m := &MapOf[K, V]{
        shards:    make([]*mapShard[K, V], shardCount),
        shardMask: uint64(shardCount - 1),
    }

    for i := 0; i < shardCount; i++ {
        m.shards[i] = &mapShard[K, V]{
            data: make(map[K]V),
        }
    }

    return m
}

func (m *MapOf[K, V]) getShard(key K) *mapShard[K, V] {
    hash := m.hasher(key)
    return m.shards[hash&m.shardMask]
}

func (m *MapOf[K, V]) Store(key K, value V) {
    shard := m.getShard(key)

    shard.mu.Lock()
    shard.data[key] = value
    shard.mu.Unlock()
}

func (m *MapOf[K, V]) Load(key K) (V, bool) {
    shard := m.getShard(key)

    shard.mu.RLock()
    val, ok := shard.data[key]
    shard.mu.RUnlock()

    return val, ok
}
```

#### sync.Map과 비교

```go
package main

import (
    "fmt"
    "sync"
    "testing"
    "github.com/puzpuzpuz/xsync/v3"
)

// sync.Map (표준 라이브러리)
func BenchmarkSyncMap(b *testing.B) {
    m := &sync.Map{}

    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Store(i, i)
            m.Load(i)
            i++
        }
    })
}

// xsync.MapOf
func BenchmarkXsyncMap(b *testing.B) {
    m := xsync.NewMapOf[int, int]()

    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            m.Store(i, i)
            m.Load(i)
            i++
        }
    })
}

// 결과 (대략적):
// BenchmarkSyncMap-8     5000000    250 ns/op
// BenchmarkXsyncMap-8   10000000    120 ns/op (2배 빠름!)
```

---

### 2. MPMCQueue - Lock-Free Queue

#### 개요
Bounded, lock-free MPMC (Multi-Producer Multi-Consumer) queue입니다.

#### 사용법

```go
package main

import (
    "fmt"
    "github.com/puzpuzpuz/xsync/v3"
)

func main() {
    // 용량 지정 (2의 제곱수 권장)
    queue := xsync.NewMPMCQueue[int](1024)

    // Enqueue (Non-blocking)
    ok := queue.TryEnqueue(42)
    if !ok {
        fmt.Println("Queue is full")
    }

    // Dequeue (Non-blocking)
    val, ok := queue.TryDequeue()
    if ok {
        fmt.Println("Dequeued:", val)
    } else {
        fmt.Println("Queue is empty")
    }
}
```

#### Producer-Consumer 패턴

```go
package main

import (
    "fmt"
    "sync"
    "github.com/puzpuzpuz/xsync/v3"
)

func main() {
    queue := xsync.NewMPMCQueue[int](1024)
    var wg sync.WaitGroup

    // 4 Producers
    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()

            for j := 0; j < 1000; j++ {
                for !queue.TryEnqueue(id*1000 + j) {
                    // 큐가 꽉 찬 경우 재시도
                }
            }
        }(i)
    }

    // 4 Consumers
    results := make([]int, 0, 4000)
    var resultMu sync.Mutex

    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            for j := 0; j < 1000; j++ {
                var val int
                var ok bool

                for {
                    val, ok = queue.TryDequeue()
                    if ok {
                        break
                    }
                    // 큐가 비어있으면 재시도
                }

                resultMu.Lock()
                results = append(results, val)
                resultMu.Unlock()
            }
        }()
    }

    wg.Wait()
    fmt.Println("Processed:", len(results), "items")
}
```

#### 내부 구조 (간소화)

```go
// Ring buffer 기반 구현
type MPMCQueue[T any] struct {
    _         [8]uint64  // padding
    head      uint64     // Consumer index
    _         [8]uint64  // padding (false sharing 방지)
    tail      uint64     // Producer index
    _         [8]uint64  // padding
    capacity  uint64
    mask      uint64
    slots     []slot[T]
}

type slot[T any] struct {
    turn  uint64   // Sequence number
    value T
}

func NewMPMCQueue[T any](capacity int) *MPMCQueue[T] {
    // capacity를 2의 제곱수로 반올림
    capacity = roundUpPowerOfTwo(capacity)

    q := &MPMCQueue[T]{
        capacity: uint64(capacity),
        mask:     uint64(capacity - 1),
        slots:    make([]slot[T], capacity),
    }

    // 각 슬롯 초기화
    for i := 0; i < capacity; i++ {
        q.slots[i].turn = uint64(i)
    }

    return q
}

func (q *MPMCQueue[T]) TryEnqueue(value T) bool {
    for {
        // Tail 읽기
        tail := atomic.LoadUint64(&q.tail)
        slot := &q.slots[tail&q.mask]
        turn := atomic.LoadUint64(&slot.turn)

        if turn == tail {
            // 슬롯 사용 가능
            if atomic.CompareAndSwapUint64(&q.tail, tail, tail+1) {
                slot.value = value
                atomic.StoreUint64(&slot.turn, tail+1)
                return true
            }
        } else if turn < tail {
            // 큐가 꽉 참
            return false
        }
        // 다른 스레드가 처리 중, 재시도
    }
}

func (q *MPMCQueue[T]) TryDequeue() (T, bool) {
    var zero T

    for {
        // Head 읽기
        head := atomic.LoadUint64(&q.head)
        slot := &q.slots[head&q.mask]
        turn := atomic.LoadUint64(&slot.turn)

        if turn == head+1 {
            // 데이터 사용 가능
            if atomic.CompareAndSwapUint64(&q.head, head, head+1) {
                value := slot.value
                atomic.StoreUint64(&slot.turn, head+q.capacity)
                return value, true
            }
        } else if turn < head+1 {
            // 큐가 비어있음
            return zero, false
        }
        // 다른 스레드가 처리 중, 재시도
    }
}
```

---

### 3. RBMutex - Range-Based Mutex

#### 개요
범위 기반으로 락을 획득하는 특수한 Mutex입니다.

#### 사용 사례

```go
package main

import (
    "github.com/puzpuzpuz/xsync/v3"
)

// 파일의 특정 범위만 락
type FileLocker struct {
    rbmu *xsync.RBMutex
}

func NewFileLocker() *FileLocker {
    return &FileLocker{
        rbmu: xsync.NewRBMutex(),
    }
}

func (fl *FileLocker) LockRange(start, end int64) func() {
    // 특정 범위만 락
    token := fl.rbmu.Lock(start, end)

    // Unlock 함수 반환
    return func() {
        fl.rbmu.Unlock(token)
    }
}

func main() {
    locker := NewFileLocker()

    // Thread 1: 0-100 범위 락
    unlock1 := locker.LockRange(0, 100)
    // ... 작업 ...
    unlock1()

    // Thread 2: 200-300 범위 락 (동시 가능!)
    unlock2 := locker.LockRange(200, 300)
    // ... 작업 ...
    unlock2()

    // Thread 3: 50-150 범위 락 (Thread 1과 겹치므로 대기)
    unlock3 := locker.LockRange(50, 150)
    // ... 작업 ...
    unlock3()
}
```

---

### 4. Counter - 고성능 카운터

#### 개요
분산 카운터로 락 경합을 최소화합니다.

#### 사용법

```go
package main

import (
    "fmt"
    "sync"
    "github.com/puzpuzpuz/xsync/v3"
)

func main() {
    counter := xsync.NewCounter()

    var wg sync.WaitGroup

    // 여러 고루틴에서 동시에 증가
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()

            for j := 0; j < 1000; j++ {
                counter.Inc()  // 매우 빠름!
            }
        }()
    }

    wg.Wait()

    fmt.Println("Total:", counter.Value())  // 100000
}
```

#### 내부 구조

```go
// CPU 별로 카운터 분산
type Counter struct {
    shards []*counterShard
}

type counterShard struct {
    _     [8]uint64  // padding
    value int64
    _     [8]uint64  // padding
}

func (c *Counter) Inc() {
    // 현재 CPU의 샤드에 증가
    shard := c.getShard()
    atomic.AddInt64(&shard.value, 1)
}

func (c *Counter) Value() int64 {
    // 모든 샤드의 합
    var sum int64
    for _, shard := range c.shards {
        sum += atomic.LoadInt64(&shard.value)
    }
    return sum
}
```

---

## 🚀 실전 예제

### 캐시 구현

```go
package main

import (
    "fmt"
    "sync"
    "time"
    "github.com/puzpuzpuz/xsync/v3"
)

type CacheEntry[V any] struct {
    value     V
    expiresAt time.Time
}

type Cache[K comparable, V any] struct {
    data    *xsync.MapOf[K, *CacheEntry[V]]
    ttl     time.Duration
    stopCh  chan struct{}
    wg      sync.WaitGroup
}

func NewCache[K comparable, V any](ttl time.Duration) *Cache[K, V] {
    c := &Cache[K, V]{
        data:   xsync.NewMapOf[K, *CacheEntry[V]](),
        ttl:    ttl,
        stopCh: make(chan struct{}),
    }

    // 만료된 항목 정리 고루틴
    c.wg.Add(1)
    go c.cleanupLoop()

    return c
}

func (c *Cache[K, V]) Set(key K, value V) {
    entry := &CacheEntry[V]{
        value:     value,
        expiresAt: time.Now().Add(c.ttl),
    }
    c.data.Store(key, entry)
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    entry, ok := c.data.Load(key)
    if !ok {
        var zero V
        return zero, false
    }

    if time.Now().After(entry.expiresAt) {
        // 만료됨
        c.data.Delete(key)
        var zero V
        return zero, false
    }

    return entry.value, true
}

func (c *Cache[K, V]) Delete(key K) {
    c.data.Delete(key)
}

func (c *Cache[K, V]) cleanupLoop() {
    defer c.wg.Done()

    ticker := time.NewTicker(time.Minute)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            now := time.Now()
            c.data.Range(func(key K, entry *CacheEntry[V]) bool {
                if now.After(entry.expiresAt) {
                    c.data.Delete(key)
                }
                return true
            })

        case <-c.stopCh:
            return
        }
    }
}

func (c *Cache[K, V]) Close() {
    close(c.stopCh)
    c.wg.Wait()
}

func main() {
    cache := NewCache[string, int](5 * time.Second)
    defer cache.Close()

    cache.Set("key1", 42)

    val, ok := cache.Get("key1")
    fmt.Println("key1:", val, ok)  // key1: 42 true

    time.Sleep(6 * time.Second)

    val, ok = cache.Get("key1")
    fmt.Println("key1:", val, ok)  // key1: 0 false (만료됨)
}
```

### Work Pool

```go
package main

import (
    "fmt"
    "sync"
    "github.com/puzpuzpuz/xsync/v3"
)

type WorkPool[T any] struct {
    queue   *xsync.MPMCQueue[T]
    workers int
    handler func(T)
    wg      sync.WaitGroup
    stopCh  chan struct{}
}

func NewWorkPool[T any](workers, queueSize int, handler func(T)) *WorkPool[T] {
    wp := &WorkPool[T]{
        queue:   xsync.NewMPMCQueue[T](queueSize),
        workers: workers,
        handler: handler,
        stopCh:  make(chan struct{}),
    }

    // Worker 고루틴 시작
    for i := 0; i < workers; i++ {
        wp.wg.Add(1)
        go wp.worker()
    }

    return wp
}

func (wp *WorkPool[T]) Submit(task T) bool {
    return wp.queue.TryEnqueue(task)
}

func (wp *WorkPool[T]) worker() {
    defer wp.wg.Done()

    for {
        select {
        case <-wp.stopCh:
            return
        default:
            task, ok := wp.queue.TryDequeue()
            if ok {
                wp.handler(task)
            }
        }
    }
}

func (wp *WorkPool[T]) Stop() {
    close(wp.stopCh)
    wp.wg.Wait()
}

func main() {
    pool := NewWorkPool(4, 1024, func(task int) {
        fmt.Println("Processing:", task)
    })

    for i := 0; i < 100; i++ {
        pool.Submit(i)
    }

    pool.Stop()
}
```

---

## 📊 성능 비교

### MapOf vs sync.Map

```
작업: 50% Read, 50% Write
스레드: 8

sync.Map:        250 ns/op
xsync.MapOf:     120 ns/op  (2배 빠름)

작업: 90% Read, 10% Write
sync.Map:        180 ns/op
xsync.MapOf:      80 ns/op  (2.25배 빠름)
```

### MPMCQueue vs channel

```
작업: Enqueue + Dequeue
스레드: 8

channel:          350 ns/op
xsync.MPMCQueue:  80 ns/op   (4배 빠름)
```

---

## 🎓 학습 포인트

### 1. 설치

```bash
go get github.com/puzpuzpuz/xsync/v3
```

### 2. 간단한 예제

```go
package main

import (
    "fmt"
    "github.com/puzpuzpuz/xsync/v3"
)

func main() {
    // Map
    m := xsync.NewMapOf[string, int]()
    m.Store("key", 42)
    val, _ := m.Load("key")
    fmt.Println(val)

    // Queue
    q := xsync.NewMPMCQueue[int](1024)
    q.TryEnqueue(42)
    val2, _ := q.TryDequeue()
    fmt.Println(val2)

    // Counter
    c := xsync.NewCounter()
    c.Inc()
    fmt.Println(c.Value())
}
```

---

## ⚠️ 주의사항

1. **MPMCQueue 용량**: 2의 제곱수로 설정
2. **Range 메서드**: 스냅샷이 아니므로 일관성 보장 없음
3. **Size 메서드**: 대략적인 값 (정확하지 않음)

---

## 🔗 관련 리소스

- GitHub: https://github.com/puzpuzpuz/xsync
- Go Generics: https://go.dev/doc/tutorial/generics

---

## 📚 다음 단계

1. **xsync 설치 및 예제 실행**
2. **자신의 프로젝트에 적용**
3. **다음: [추천 레포지토리](./06-recommended-repos.md)**

---

*이 문서는 학습 목적으로 작성되었습니다.*
