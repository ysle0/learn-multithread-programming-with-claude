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

## Internal Mechanisms

### Channel 구조체 (runtime.hchan)

```go
// runtime/chan.go
type hchan struct {
    qcount   uint           // 버퍼 내 요소 개수
    dataqsiz uint           // 버퍼 크기 (make의 두 번째 인수)
    buf      unsafe.Pointer // 순환 버퍼 포인터
    elemsize uint16         // 요소 크기
    closed   uint32         // 닫힘 플래그 (1 = closed)
    elemtype *_type         // 요소 타입
    sendx    uint           // 다음 send 위치
    recvx    uint           // 다음 receive 위치
    recvq    waitq          // 수신 대기 goroutine 큐
    sendq    waitq          // 송신 대기 goroutine 큐
    lock     mutex          // 채널 락
}

// 대기 큐 노드
type sudog struct {
    g     *g           // 대기 중인 goroutine
    elem  unsafe.Pointer // 데이터 포인터
    // ... 기타 필드
}
```

### 메모리 레이아웃

```
Unbuffered Channel (make(chan T)):
┌─────────────────────────────────────────┐
│  hchan                                  │
│  ┌───────────────────────────────────┐ │
│  │ qcount = 0, dataqsiz = 0         │ │
│  │ buf = nil                         │ │
│  │ recvq: [sudog] → [sudog] → ...   │ │
│  │ sendq: [sudog] → [sudog] → ...   │ │
│  └───────────────────────────────────┘ │
└─────────────────────────────────────────┘

Buffered Channel (make(chan T, 3)):
┌─────────────────────────────────────────┐
│  hchan                                  │
│  ┌───────────────────────────────────┐ │
│  │ qcount = 2, dataqsiz = 3         │ │
│  │ buf → ┌───┬───┬───┐              │ │
│  │       │ A │ B │   │              │ │
│  │       └───┴───┴───┘              │ │
│  │         ↑       ↑                 │ │
│  │      recvx   sendx                │ │
│  └───────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

### Send 연산 (ch <- value)

```
ch <- value 실행 흐름:

1. 채널 락 획득
     │
     ▼
2. 수신 대기자 있음? (recvq 비어있지 않음)
     │
     ├─ YES: 직접 전달
     │   - recvq에서 sudog 팝
     │   - 값을 sudog.elem에 직접 복사
     │   - 수신자 goroutine 깨우기 (goready)
     │   - return
     │
     └─ NO: 버퍼에 공간 있음? (qcount < dataqsiz)
           │
           ├─ YES: 버퍼에 저장
           │   - buf[sendx]에 값 복사
           │   - sendx++, qcount++
           │   - return
           │
           └─ NO: 블로킹
               - 현재 G를 sudog로 래핑
               - sendq에 추가
               - gopark() (G를 대기 상태로)
               - 깨어나면 return
```

### Receive 연산 (value = <-ch)

```
value = <-ch 실행 흐름:

1. 채널 락 획득
     │
     ▼
2. 송신 대기자 있음? (sendq 비어있지 않음)
     │
     ├─ YES (버퍼 있는 경우):
     │   - 버퍼 head에서 값 가져오기
     │   - 송신자의 값을 버퍼 tail에 저장
     │   - 송신자 깨우기
     │
     ├─ YES (버퍼 없는 경우):
     │   - 송신자의 sudog에서 직접 값 복사
     │   - 송신자 깨우기
     │
     └─ NO: 버퍼에 데이터 있음?
           │
           ├─ YES: 버퍼에서 읽기
           │   - buf[recvx]에서 값 복사
           │   - recvx++, qcount--
           │
           └─ NO: 블로킹
               - 현재 G를 sudog로 래핑
               - recvq에 추가
               - gopark()
```

### 직접 전달 최적화 (Direct Send)

Unbuffered 채널에서 송신자와 수신자가 만나면 버퍼 복사 없이 직접 전달:

```go
// 송신자 스택 → 수신자 스택 직접 복사
func send(c *hchan, sg *sudog, ep unsafe.Pointer) {
    // sg.elem: 수신자의 변수 주소
    // ep: 송신할 값의 주소
    memmove(sg.elem, ep, c.elemtype.size)
}

// 컨텍스트 스위칭 비용 없이 데이터 전달
// → Unbuffered 채널이 동기화 포인트로 효율적
```

### Close 연산

```go
func closechan(c *hchan) {
    lock(&c.lock)

    c.closed = 1  // closed 플래그 설정

    // 모든 수신 대기자 깨우기 (zero value 반환)
    for {
        sg := c.recvq.dequeue()
        if sg == nil {
            break
        }
        sg.elem = nil  // zero value
        goready(sg.g)
    }

    // 모든 송신 대기자 깨우기 (panic 발생)
    for {
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }
        goready(sg.g)  // 깨어나면 panic
    }

    unlock(&c.lock)
}
```

### nil 채널 동작

```go
var ch chan int  // nil

ch <- 1   // 영원히 블록 (deadlock이 아님)
<-ch      // 영원히 블록
close(ch) // panic!

// select에서 nil 채널은 해당 case 비활성화에 유용
select {
case <-ch:     // ch가 nil이면 이 case는 절대 선택 안 됨
case <-other:  // other만 체크
}
```

## Navigation

- [Back to Go Overview](./README.md)
- Previous: [Goroutines](./01-goroutine.md)
- Next: [Select Statement](./03-select.md)
