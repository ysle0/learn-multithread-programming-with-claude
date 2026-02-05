# Go의 Select 문

`select` 문은 goroutine이 여러 channel 연산을 동시에 대기하면서, 먼저 준비된 것을 선택할 수 있게 해줍니다.

## 기본 Select

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

## Default Case (비차단)

```go
select {
case msg := <-ch:
    fmt.Println(msg)
default:
    fmt.Println("No message")
}
```

## 타임아웃 패턴

```go
select {
case result := <-ch:
    fmt.Println(result)
case <-time.After(time.Second):
    fmt.Println("Timeout!")
}
```

## 전체 예제

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

## 내부 메커니즘

### Select 구조체 (runtime.scase)

```go
// runtime/select.go
type scase struct {
    c    *hchan         // 채널
    elem unsafe.Pointer  // 데이터 포인터
    kind uint16         // caseSend, caseRecv, caseDefault
    // ...
}

// select 컴파일 결과
// 컴파일러가 select를 selectgo() 호출로 변환
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool)
```

### Select 실행 알고리즘

```
selectgo() 실행 흐름:

1. 랜덤 순서 결정 (pollorder)
   ┌─────────────────────────────────────────┐
   │ cases: [0, 1, 2, 3]                     │
   │ pollorder: [2, 0, 3, 1] (셔플됨)       │
   │                                         │
   │ → 공정성 보장 (특정 case에 편향 방지)   │
   └─────────────────────────────────────────┘

2. 락 순서 결정 (lockorder)
   ┌─────────────────────────────────────────┐
   │ 채널 주소순으로 정렬                     │
   │ → 데드락 방지 (항상 같은 순서로 락)     │
   │                                         │
   │ lockorder: [ch_addr1, ch_addr2, ...]   │
   └─────────────────────────────────────────┘

3. 모든 채널 락 획득 (lockorder 순서)

4. 준비된 case 확인 (pollorder 순서)
   ┌─────────────────────────────────────────┐
   │ for _, i := range pollorder:           │
   │     if case[i] is ready:               │
   │         unlock all, execute case       │
   │         return                         │
   └─────────────────────────────────────────┘

5. default case 있으면 실행

6. 없으면 대기:
   ┌─────────────────────────────────────────┐
   │ 현재 G를 모든 채널의 대기 큐에 등록     │
   │ (각 채널에 sudog 추가)                  │
   │                                         │
   │ gopark() → 대기 상태로 전환             │
   │                                         │
   │ (채널 연산 발생 시 깨어남)              │
   │                                         │
   │ 다른 채널 대기 큐에서 자신 제거         │
   │ 선택된 case 실행                        │
   └─────────────────────────────────────────┘
```

### 랜덤 선택의 이유

```go
// 여러 case가 동시에 ready일 때 랜덤 선택
select {
case <-ch1:  // 50% 확률
case <-ch2:  // 50% 확률
}

// 이유:
// 1. 공정성: 특정 채널에 편향되지 않음
// 2. 기아 방지: 모든 채널이 공평하게 선택됨
// 3. 예측 불가: 프로그램 동작이 결정적이지 않음
//    → 동시성 버그 발견에 도움

// 랜덤 시드: runtime.fastrand()
// → 저비용 의사 난수 생성기
```

### 컴파일러 최적화

```go
// 단일 case + default → 비블로킹 연산으로 변환
select {
case v := <-ch:
    use(v)
default:
    noData()
}

// 컴파일러가 다음으로 변환:
if v, ok := chanrecv(ch, nonblocking); ok {
    use(v)
} else {
    noData()
}
```

**빈 select 처리**:
```go
select {}  // 영원히 블록
// → gopark(nil, nil, "select (no cases)")
```

### time.After 메모리 누수

```go
// 문제: 루프에서 time.After 사용
for {
    select {
    case <-ch:
        // 처리
    case <-time.After(time.Second):  // 매번 새 Timer 생성!
        // 타임아웃
    }
}

// 해결: time.Timer 재사용
timer := time.NewTimer(time.Second)
for {
    select {
    case <-ch:
        if !timer.Stop() {
            <-timer.C
        }
        timer.Reset(time.Second)
    case <-timer.C:
        timer.Reset(time.Second)
    }
}
```

## 탐색

- [Go 개요로 돌아가기](./README.md)
- 이전: [Channel](./02-channel.md)
- 다음: [Sync 패키지](./04-sync-package.md)
