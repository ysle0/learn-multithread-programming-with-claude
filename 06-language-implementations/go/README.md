# Go 동시성 프로그래밍

Go는 동시성을 핵심 기능으로 처음부터 설계되었습니다. 경량 goroutine과 channel을 통해 동시성 프로그래밍을 자연스럽고 효율적으로 만들어줍니다.

## 개요

Go의 동시성 모델은 CSP(Communicating Sequential Processes)를 기반으로 합니다:
- **Goroutine**: Go 런타임이 관리하는 경량 스레드
- **Channel**: goroutine 간 타입 안전한 통신
- **Select**: channel 연산 다중화
- **Sync 패키지**: 전통적인 동기화 기본 요소
- **Context**: 취소 및 데드라인

## 핵심 구성 요소

### 1. [Goroutine](./01-goroutine.md)
- `go` 키워드로 goroutine 생성
- 경량 동시성
- 런타임 스케줄러 (M:N 모델)
- Goroutine 수명 주기

### 2. [Channel](./02-channel.md)
- Unbuffered 및 buffered channel
- 송신과 수신
- Channel 닫기
- Channel에 대한 range 반복

### 3. [Select 문](./03-select.md)
- Channel 연산 다중화
- 비차단 연산
- 타임아웃과 기본값
- Select 패턴

### 4. [Sync 패키지](./04-sync-package.md)
- `sync.Mutex`와 `sync.RWMutex`
- `sync.WaitGroup`
- `sync.Once`
- `sync.Pool`

### 5. [Context 패키지](./05-context.md)
- 취소 전파
- 타임아웃과 데드라인
- 요청 범위 값
- Context 모범 사례

## 다른 언어와의 간략 비교

| 기능 | Go | 비교 |
|---------|-----|------------|
| **동시성 모델** | Goroutine + Channel (CSP) | 동시성 프로그래밍에 가장 자연스러움 |
| **스레드 무게** | goroutine당 ~2KB | 가장 가벼움 - 수백만 개 가능 |
| **통신** | Channel (내장) | 가장 우아한 메시지 전달 |
| **문법** | `go` 키워드 | 동시성 작업 생성이 가장 간단 |
| **학습 곡선** | 완만함 | C++보다 쉽고 async/await보다 단순 |

## 핵심 원칙

### 1. Goroutine은 저렴하다

```go
// 수백만 개의 goroutine을 쉽게 생성 가능
for i := 0; i < 1000000; i++ {
    go func(n int) {
        // 작업
    }(i)
}
```

### 2. 통신을 통해 메모리를 공유하라

```go
// 좋음: channel을 사용하여 통신
ch := make(chan int)
go func() {
    ch <- 42  // 송신
}()
result := <-ch  // 수신

// 덜 선호됨: mutex를 사용한 공유 메모리
var mu sync.Mutex
var shared int
mu.Lock()
shared = 42
mu.Unlock()
```

### 3. 공유 메모리로 통신하지 마라

Go의 철학: "공유 메모리로 통신하지 말고, 통신을 통해 메모리를 공유하라."

## 일반적인 패턴

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

// 사용법
for result := range square(generator()) {
    fmt.Println(result)
}
```

## 모범 사례

1. **항상 goroutine 완료를 처리하라**: WaitGroup이나 channel 사용
2. **송신자 쪽에서 channel을 닫아라**: 수신자는 절대 닫지 말 것
3. **타임아웃에 select를 사용하라**: 무한정 차단하지 말 것
4. **취소에는 context를 전달하라**: 함수의 첫 번째 매개변수로
5. **닫힌 channel을 확인하라**: 수신 연산을 테스트할 것
6. **goroutine 누수를 방지하라**: 모든 goroutine이 종료할 수 있도록 보장
7. **buffered channel을 현명하게 사용하라**: 알려진 용량에 대해서만

## 일반적인 함정

### 1. Goroutine 누수

```go
// 나쁨: goroutine이 절대 종료되지 않음
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // 아무것도 송신되지 않으면 영원히 차단됨
    }()
}

// 좋음: 타임아웃 사용
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

### 2. 수신자가 Channel 닫기

```go
// 나쁨: 수신자가 닫음
go func() {
    for val := range ch {
        process(val)
    }
    close(ch)  // 잘못됨!
}()

// 좋음: 송신자가 닫음
go func() {
    for i := 0; i < 10; i++ {
        ch <- i
    }
    close(ch)  // 올바름
}()
```

### 3. 루프 변수 캡처

```go
// 나쁨: 루프 변수를 캡처함
for i := 0; i < 10; i++ {
    go func() {
        fmt.Println(i)  // 잘못된 값을 출력할 수 있음
    }()
}

// 좋음: 매개변수로 전달
for i := 0; i < 10; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

### 4. 닫힌 Channel에 송신

```go
// 나쁨: 패닉 발생
ch := make(chan int)
close(ch)
ch <- 1  // 패닉!

// 좋음: 닫기 전 확인
var once sync.Once
once.Do(func() {
    close(ch)
})
```

### 5. 경쟁 조건

```go
// 나쁨: 데이터 경쟁
var counter int
for i := 0; i < 1000; i++ {
    go func() {
        counter++  // 경쟁!
    }()
}

// 좋음: atomic이나 mutex 사용
var counter int64
for i := 0; i < 1000; i++ {
    go func() {
        atomic.AddInt64(&counter, 1)
    }()
}
```

## 성능 특성

### Goroutine 오버헤드
- **생성**: ~1 마이크로초
- **메모리**: ~2KB 초기 스택 (증가 가능)
- **컨텍스트 전환**: 매우 빠름 (프로세스 내)
- **확장성**: 수백만 개의 goroutine 가능

### Channel 성능
- **Unbuffered**: 동기화 지점, 더 느림
- **Buffered**: 더 빠름, 송신자/수신자 분리
- **송신/수신**: ~100 나노초

## 경쟁 감지

```bash
# 경쟁 감지기로 실행
go run -race main.go

# 경쟁 감지기로 빌드
go build -race

# 경쟁 감지기로 테스트
go test -race
```

## 도구 및 디버깅

### 프로파일링을 위한 pprof

```go
import _ "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    // 프로그램 코드
}

// http://localhost:6060/debug/pprof/ 에서 접근
```

### Goroutine 프로파일링

```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine
```

### 추적

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// 코드
```

## 최신 Go 기능

### 제네릭 (Go 1.18+)

```go
func SendSlice[T any](ch chan<- T, slice []T) {
    for _, v := range slice {
        ch <- v
    }
    close(ch)
}
```

### Goroutine을 사용한 에러 처리

```go
type Result struct {
    Value int
    Error error
}

func compute() Result {
    ch := make(chan Result, 1)
    go func() {
        // 작업 수행
        ch <- Result{Value: 42}
    }()
    return <-ch
}
```

## 추가 참고 자료

- **Go Concurrency Patterns** by Rob Pike
- **Concurrency in Go** by Katherine Cox-Buday
- Go 블로그: https://go.dev/blog/
- Effective Go: https://go.dev/doc/effective_go

## 탐색

- [언어 구현으로 돌아가기](../)
- 다음 주제:
  - [Goroutine](./01-goroutine.md)
  - [Channel](./02-channel.md)
  - [Select 문](./03-select.md)
  - [Sync 패키지](./04-sync-package.md)
  - [Context 패키지](./05-context.md)
