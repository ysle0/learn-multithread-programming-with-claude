# Go의 Channel

Channel은 goroutine 간에 통신하고 동기화하는 Go의 방식입니다. goroutine 간에 값을 전달하기 위한 타입 안전한 통로입니다.

## 기본 개념

### Channel 생성

```go
ch := make(chan int)           // Unbuffered
ch := make(chan int, 5)        // Buffered (용량 5)
ch := make(chan string, 100)   // Buffered string channel
```

### 송신과 수신

```go
ch := make(chan int)

// 송신
go func() {
    ch <- 42  // channel에 42 송신
}()

// 수신
value := <-ch  // channel에서 수신

// 수신 후 버림
<-ch
```

## Unbuffered Channel

### 동기화

```go
func main() {
    ch := make(chan string)

    go func() {
        ch <- "hello"  // 수신될 때까지 차단
    }()

    msg := <-ch  // 송신될 때까지 차단
    fmt.Println(msg)
}
```

## Buffered Channel

### 비차단 송신 (가득 찰 때까지)

```go
ch := make(chan int, 3)

ch <- 1  // 차단 안 됨
ch <- 2  // 차단 안 됨
ch <- 3  // 차단 안 됨
// ch <- 4  // 차단됨 (버퍼 가득 참)

fmt.Println(<-ch)  // 1
fmt.Println(<-ch)  // 2
```

## Channel 닫기

### 송신자가 닫기

```go
func sender(ch chan<- int) {
    for i := 0; i < 5; i++ {
        ch <- i
    }
    close(ch)  // 송신자가 닫음
}

func main() {
    ch := make(chan int)
    go sender(ch)

    // 닫힐 때까지 수신
    for val := range ch {
        fmt.Println(val)
    }
}
```

### 닫힘 여부 확인

```go
val, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}
```

## 방향 (송신 전용, 수신 전용)

```go
// 송신 전용 channel
func sender(ch chan<- int) {
    ch <- 42
    // val := <-ch  // 컴파일 에러
}

// 수신 전용 channel
func receiver(ch <-chan int) {
    val := <-ch
    // ch <- 42  // 컴파일 에러
}
```

## 일반적인 패턴

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

    // 워커 시작
    for w := 0; w < 3; w++ {
        go worker(jobs, results)
    }

    // 작업 송신
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)

    // 결과 수신
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

## 내부 메커니즘

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

## 탐색

- [Go 개요로 돌아가기](./README.md)
- 이전: [Goroutine](./01-goroutine.md)
- 다음: [Select 문](./03-select.md)
