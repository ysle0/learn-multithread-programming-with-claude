# Go의 Goroutine

Goroutine은 Go 런타임이 관리하는 경량 스레드입니다. Go 동시성 모델의 기반이 됩니다.

## 목차
- [기본 개념](#기본-개념)
- [Goroutine 생성](#goroutine-생성)
- [Goroutine 수명 주기](#goroutine-수명-주기)
- [WaitGroup](#waitgroup)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 함정](#일반적인-함정)

## 기본 개념

### Goroutine이란?

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
    go hello()  // goroutine 시작

    time.Sleep(time.Second)  // 대기 (이상적이지 않음, WaitGroup 참조)
    fmt.Println("Main function")
}
```

### Goroutine 동작 원리

- **M:N 스케줄러**: 많은 goroutine을 더 적은 OS 스레드에서 실행
- **초기 스택**: ~2KB (자동으로 증가/축소)
- **협력적**: goroutine은 함수 호출, channel 연산 등에서 양보
- **저렴함**: 수백만 개 생성 가능

## Goroutine 생성

### 익명 함수

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    // 익명 함수로 시작
    go func() {
        fmt.Println("Anonymous goroutine")
    }()

    // 매개변수 사용
    message := "Hello"
    go func(msg string) {
        fmt.Println(msg)
    }(message)

    time.Sleep(time.Second)
}
```

### 명명된 함수

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

### 메서드를 Goroutine으로 실행

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

## Goroutine 수명 주기

### 간단한 Goroutine

```go
func main() {
    done := make(chan bool)

    go func() {
        fmt.Println("Goroutine running")
        time.Sleep(time.Second)
        fmt.Println("Goroutine done")
        done <- true
    }()

    <-done  // goroutine 대기
    fmt.Println("Main done")
}
```

## WaitGroup

### 기본 WaitGroup

```go
import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()  // 완료 시 카운터 감소
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)  // 카운터 증가
        go worker(i, &wg)
    }

    wg.Wait()  // 카운터가 0이 될 때까지 차단
    fmt.Println("All workers done")
}
```

### 에러 처리가 있는 WaitGroup

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

## 다른 언어와의 비교

### Go vs. C++
```go
// Go: 매우 가벼움
go doWork()

// C++ 동등 코드:
// std::thread t(doWork);
// t.join();
// (훨씬 무거움 - OS 스레드)
```

### Go vs. C#
```go
// Go: Goroutine
go doWork()

// C# 동등 코드:
// Task.Run(() => doWork());
// (Task는 스레드 풀 사용, 여전히 더 무거움)
```

## 모범 사례

### 1. 항상 Goroutine 완료를 조율하라

```go
// 좋음: WaitGroup 사용
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()
wg.Wait()

// 좋음: channel 사용
done := make(chan bool)
go func() {
    work()
    done <- true
}()
<-done
```

### 2. 대기를 위해 Sleep 사용하지 마라

```go
// 나쁨: 대기를 위해 sleep 사용
go work()
time.Sleep(time.Second)

// 좋음: 동기화 사용
done := make(chan bool)
go func() {
    work()
    done <- true
}()
<-done
```

### 3. 매개변수를 통해 데이터를 전달하라

```go
// 좋음: 매개변수로 전달
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}

// 나쁨: 루프 변수 캡처
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // 잘못된 값을 출력할 수 있음
    }()
}
```

## 일반적인 함정

### 루프 변수 캡처

```go
// 나쁨: 모든 goroutine이 같은 변수를 봄
for _, val := range values {
    go func() {
        fmt.Println(val)  // 잘못됨!
    }()
}

// 좋음: 매개변수로 전달
for _, val := range values {
    go func(v string) {
        fmt.Println(v)
    }(val)
}
```

### Goroutine 누수

```go
// 나쁨: channel이 수신하지 않으면 goroutine 누수
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // 영원히 차단됨
    }()
}

// 좋음: 타임아웃 사용
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

## 내부 메커니즘

### G-M-P 스케줄러 모델

Go 런타임의 스케줄러는 M:N 스레딩 모델을 사용합니다:

```
G (Goroutine): 실행할 코드 + 스택 + 상태
M (Machine):   OS 스레드
P (Processor): 스케줄링 컨텍스트 (로컬 런큐 포함)

┌─────────────────────────────────────────────────────────────┐
│                         Scheduler                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │              Global Run Queue (GRQ)                 │  │
│  │    [G] [G] [G] ... (대기 중인 goroutine)             │  │
│  └─────────────────────────────────────────────────────┘  │
│                           │                                 │
│          ┌────────────────┼────────────────┐               │
│          ▼                ▼                ▼               │
│    ┌─────────┐      ┌─────────┐      ┌─────────┐          │
│    │   P₀    │      │   P₁    │      │   P₂    │          │
│    │ LRQ:    │      │ LRQ:    │      │ LRQ:    │          │
│    │ [G][G]  │      │ [G]     │      │ [G][G]  │          │
│    │         │      │         │      │[G]      │          │
│    └────┬────┘      └────┬────┘      └────┬────┘          │
│         │                │                │                │
│         ▼                ▼                ▼                │
│    ┌─────────┐      ┌─────────┐      ┌─────────┐          │
│    │   M₀    │      │   M₁    │      │   M₂    │          │
│    │(running │      │(running │      │(running │          │
│    │   G)    │      │   G)    │      │   G)    │          │
│    └─────────┘      └─────────┘      └─────────┘          │
│         ↓                ↓                ↓                │
│    [OS Thread]      [OS Thread]      [OS Thread]          │
└─────────────────────────────────────────────────────────────┘
```

### Goroutine 구조체 (runtime.g)

```go
// runtime/runtime2.go (단순화)
type g struct {
    stack       stack    // 스택 범위 [lo, hi)
    stackguard0 uintptr  // 스택 오버플로우 체크용
    stackguard1 uintptr  // C 코드용 가드

    _panic      *_panic  // 현재 panic 상태
    _defer      *_defer  // defer 체인

    m           *m       // 현재 실행 중인 M
    sched       gobuf    // 스케줄링 컨텍스트 (PC, SP, BP, G)

    atomicstatus uint32  // goroutine 상태
    goid         int64   // goroutine ID

    // ... 기타 필드
}

// 상태 전이
const (
    _Gidle     = iota // 할당됨, 초기화 안 됨
    _Grunnable        // 실행 가능, 런큐에 있음
    _Grunning         // 실행 중 (M에 할당됨)
    _Gsyscall         // 시스템 콜 중
    _Gwaiting         // 대기 중 (채널, 락 등)
    _Gdead            // 종료됨, 재사용 가능
    _Gcopystack       // 스택 복사 중
    _Gpreempted       // 선점됨
)
```

### 스택 증가 메커니즘

Goroutine 스택은 2KB에서 시작하여 필요에 따라 자동으로 증가합니다:

```
초기 스택: 2KB
      │
      ▼
함수 호출 시 스택 체크:
┌─────────────────────────────────────┐
│  if SP < stackguard0:               │
│      call runtime.morestack()       │
└─────────────────────────────────────┘
      │
      ▼
morestack 동작:
1. 새 스택 할당 (2배 크기)
2. 기존 스택 복사 (포인터 조정 포함)
3. 기존 스택 해제
      │
      ▼
최대 스택: 1GB (64-bit), 250MB (32-bit)
```

**스택 복사 시 포인터 조정**:
```go
// 스택 내 포인터가 다른 스택 주소를 가리키면 조정
// 예: &localVar가 스택에 있으면 새 주소로 변경
// 힙 포인터는 그대로 유지

// copystack() 단순화
func copystack(gp *g, newsize uintptr) {
    old := gp.stack
    new := stackalloc(newsize)

    // 스택 내용 복사
    memmove(new.hi-used, old.hi-used, used)

    // 스택 내 포인터 조정
    adjustpointers(new, old, delta)

    gp.stack = new
    stackfree(old)
}
```

### Work Stealing 알고리즘

P의 로컬 런큐가 비면 다른 P에서 goroutine을 훔쳐옵니다:

```
M이 실행할 G를 찾는 순서:
1. 로컬 런큐 확인 (가장 빠름)
2. 글로벌 런큐 확인 (1/61 확률로 먼저 확인)
3. 네트워크 폴러 확인 (epoll/kqueue)
4. 다른 P에서 Work Stealing

Work Stealing:
┌─────────────────────────────────────────────────────────────┐
│  P₀ (비어있음)              P₁ (많은 G)                     │
│      │                         │                           │
│      └────── steal ───────────▶│                           │
│                                │                           │
│  P₀.runq = P₁.runq의 절반 가져오기                          │
└─────────────────────────────────────────────────────────────┘
```

### 선점 (Preemption)

Go 1.14+에서는 비협력적 선점(async preemption)이 도입되었습니다:

```
협력적 선점 (Go 1.13 이하):
- 함수 프롤로그에서 스택 체크
- 채널 연산, 메모리 할당 등에서 스케줄 포인트

비협력적 선점 (Go 1.14+):
- 10ms마다 sysmon이 실행 중인 G 체크
- 오래 실행 중이면 시그널 전송 (SIGURG)
- 시그널 핸들러에서 G를 preempted 상태로 변경

┌─────────────────────────────────────────────────────────────┐
│  sysmon goroutine (별도 M에서 실행)                         │
│      │                                                      │
│      ▼                                                      │
│  10ms마다 깨어남                                            │
│      │                                                      │
│      ▼                                                      │
│  오래 실행 중인 G 발견?                                      │
│      │ YES                                                  │
│      ▼                                                      │
│  preemptone(gp) → SIGURG 전송                               │
│      │                                                      │
│      ▼                                                      │
│  G가 asyncPreempt 핸들러에서 스케줄 아웃                     │
└─────────────────────────────────────────────────────────────┘
```

### Goroutine 생성 비용

```go
// newproc: 새 goroutine 생성
// 비용: ~300ns (Linux x86-64)

// 재사용 풀 (gFree):
// 종료된 goroutine은 gFree 리스트에 보관
// 새 goroutine 생성 시 풀에서 가져와 재사용
// → 메모리 할당 비용 감소

type p struct {
    // ...
    gFree struct {
        gList
        n int32  // 풀의 G 개수
    }
}
```

## 전체 예제: 동시성 웹 스크래퍼

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

## 탐색

- [Go 개요로 돌아가기](./README.md)
- 다음: [Channel](./02-channel.md)
