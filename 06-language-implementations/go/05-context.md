# Go의 Context 패키지

`context` 패키지는 API 경계를 넘어 취소 신호, 데드라인, 요청 범위 값을 전달하는 방법을 제공합니다.

## 기본 Context

### Background와 TODO

```go
import "context"

// 루트 context
ctx := context.Background()

// 어떤 context를 사용할지 모를 때
ctx := context.TODO()
```

## 취소

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
    cancel()  // goroutine 취소
    time.Sleep(time.Second)
}
```

## 타임아웃

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

## Context 값

### WithValue

```go
type key string

ctx := context.WithValue(context.Background(), key("user"), "alice")

// 값 가져오기
if user, ok := ctx.Value(key("user")).(string); ok {
    fmt.Println("User:", user)
}
```

## 모범 사례

### 함수 시그니처

```go
// 좋음: Context를 첫 번째 매개변수로
func DoWork(ctx context.Context, arg string) error {
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
        // 작업
        return nil
    }
}
```

### 항상 Context를 존중하라

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return  // 취소되면 종료
        default:
            // 작업 수행
        }
    }
}
```

## 전체 예제: 타임아웃이 있는 HTTP 요청

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

## 내부 메커니즘

### Context 인터페이스

```go
// context/context.go
type Context interface {
    // 취소 또는 타임아웃 시점 반환
    Deadline() (deadline time.Time, ok bool)

    // 취소 시 닫히는 채널
    Done() <-chan struct{}

    // Done() 후 에러 반환 (Canceled 또는 DeadlineExceeded)
    Err() error

    // 키에 해당하는 값 반환
    Value(key interface{}) interface{}
}
```

### 구현 타입들

```go
// emptyCtx: Background()와 TODO()의 기본 타입
type emptyCtx int

// cancelCtx: WithCancel()이 반환
type cancelCtx struct {
    Context
    mu       sync.Mutex
    done     chan struct{}    // lazy init
    children map[canceler]struct{}
    err      error
}

// timerCtx: WithTimeout/WithDeadline()이 반환
type timerCtx struct {
    cancelCtx
    timer    *time.Timer
    deadline time.Time
}

// valueCtx: WithValue()가 반환
type valueCtx struct {
    Context
    key, val interface{}
}
```

### Context 트리 구조

```
WithCancel, WithTimeout, WithValue를 호출하면 트리 형성:

┌─────────────────────────────────────────────────────────────┐
│                     Background()                            │
│                          │                                  │
│            ┌─────────────┼─────────────┐                   │
│            ▼             ▼             ▼                   │
│     WithTimeout    WithCancel    WithValue                 │
│     (요청 타임아웃)  (작업 취소)   (requestID)              │
│            │             │             │                   │
│            ▼             ▼             ▼                   │
│      WithValue     WithTimeout   WithCancel                │
│      (userID)      (DB 타임아웃)  (서브태스크)              │
│                          │                                  │
│                          ▼                                  │
│                    WithCancel                               │
│                    (HTTP call)                              │
└─────────────────────────────────────────────────────────────┘

취소 전파: 부모 → 자식
값 조회: 자식 → 부모 (체인 따라 올라감)
```

### 취소 전파 메커니즘

```go
// cancel() 내부 동작
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return  // 이미 취소됨
    }
    c.err = err

    // done 채널 닫기 (대기자 모두 깨움)
    if c.done == nil {
        c.done = closedchan  // 미리 닫힌 채널 재사용
    } else {
        close(c.done)
    }

    // 모든 자식 취소
    for child := range c.children {
        child.cancel(false, err)  // 재귀적 취소
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        // 부모의 children 맵에서 자신 제거
        removeChild(c.Context, c)
    }
}
```

### Done() 채널 Lazy Initialization

```go
func (c *cancelCtx) Done() <-chan struct{} {
    c.mu.Lock()
    if c.done == nil {
        c.done = make(chan struct{})  // 필요할 때만 생성
    }
    d := c.done
    c.mu.Unlock()
    return d
}

// 이유:
// 1. 메모리 절약 (Done() 호출 안 하면 채널 생성 안 함)
// 2. Background(), TODO()는 done 채널 필요 없음
```

### Value 조회 체인

```go
func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val  // 현재 노드에서 발견
    }
    return c.Context.Value(key)  // 부모로 재귀
}

// 시간 복잡도: O(n) where n = context 깊이
// 깊은 중첩 주의!

// 예시:
ctx1 := WithValue(Background(), "a", 1)
ctx2 := WithValue(ctx1, "b", 2)
ctx3 := WithValue(ctx2, "c", 3)

ctx3.Value("a")  // ctx3 → ctx2 → ctx1 순회
```

### Timer 관리 (WithTimeout/WithDeadline)

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}

func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    // 부모가 더 일찍 만료되면 부모 기준
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        return WithCancel(parent)
    }

    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    propagateCancel(parent, c)  // 부모 취소 시 자식도 취소

    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded)  // 이미 만료
        return c, func() { c.cancel(false, Canceled) }
    }

    // 타이머 설정
    c.timer = time.AfterFunc(dur, func() {
        c.cancel(true, DeadlineExceeded)
    })

    return c, func() { c.cancel(true, Canceled) }
}
```

### 메모리 누수 방지

```go
// 중요: cancel 함수는 반드시 호출해야 함!
ctx, cancel := context.WithCancel(parent)
defer cancel()  // 항상 defer로 보장

// 이유:
// 1. timerCtx의 타이머 정리
// 2. 부모의 children 맵에서 제거
// 3. goroutine 누수 방지

// 나쁜 예 (메모리 누수):
func bad() context.Context {
    ctx, _ := context.WithCancel(context.Background())
    return ctx  // cancel 함수 버림!
}
```

## 탐색

- [Go 개요로 돌아가기](./README.md)
- 이전: [Sync 패키지](./04-sync-package.md)
- [언어 구현으로 돌아가기](../)
