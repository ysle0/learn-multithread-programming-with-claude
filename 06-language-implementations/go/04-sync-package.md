# Go의 Sync 패키지

`sync` 패키지는 channel이 적합하지 않은 경우를 위한 전통적인 동기화 기본 요소를 제공합니다.

## Mutex

### 기본 Mutex

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

## RWMutex (읽기-쓰기 락)

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

## Once (한 번만 실행)

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

## sync.Map (동시성 안전 Map)

```go
var m sync.Map

// 저장
m.Store("key", "value")

// 로드
if val, ok := m.Load("key"); ok {
    fmt.Println(val)
}

// 로드 또는 저장
actual, loaded := m.LoadOrStore("key", "value")

// 삭제
m.Delete("key")

// 순회
m.Range(func(key, value interface{}) bool {
    fmt.Printf("%v: %v\n", key, value)
    return true  // 반복 계속
})
```

## 내부 메커니즘

### Mutex 구조체

```go
// sync/mutex.go
type Mutex struct {
    state int32   // 락 상태
    sema  uint32  // 세마포어 (대기자 관리)
}

// state 비트 레이아웃:
// bit 0: mutexLocked     - 락 소유 여부
// bit 1: mutexWoken      - 깨어난 대기자 존재
// bit 2: mutexStarving   - 기아 모드 여부
// bit 3+: 대기자 수 (waiter count)

const (
    mutexLocked = 1 << iota  // 1
    mutexWoken               // 2
    mutexStarving            // 4
    mutexWaiterShift = iota  // 3
)
```

### Mutex Lock 알고리즘

```
Lock() 실행 흐름:

Fast Path:
┌─────────────────────────────────────────┐
│ atomic.CompareAndSwapInt32(&m.state,    │
│                            0,           │
│                            mutexLocked) │
│ → 성공하면 즉시 return (락 획득)        │
└─────────────────────────────────────────┘
     │ 실패
     ▼
Slow Path (lockSlow):
┌─────────────────────────────────────────┐
│ 1. 스핀 시도 (멀티코어일 때)            │
│    - 4회 스핀, 매번 30 PAUSE 실행       │
│    - 락 해제되면 획득 시도              │
│                                         │
│ 2. 스핀 실패 → waiter count 증가        │
│                                         │
│ 3. semacquire(&m.sema) → 대기           │
│                                         │
│ 4. 깨어나면 락 획득 시도                │
│    - 기아 모드면 즉시 획득              │
│    - 일반 모드면 새 goroutine과 경쟁    │
└─────────────────────────────────────────┘
```

### 기아 모드 (Starvation Mode)

```
일반 모드: 새로 도착한 goroutine이 대기자보다 유리
         (이미 CPU에서 실행 중이므로)
         → 처리량 최적화, but 기아 가능

기아 모드: 1ms 이상 대기한 waiter가 있으면 전환
         → 대기자에게 직접 락 전달
         → 공정성 보장, but 처리량 감소

┌─────────────────────────────────────────┐
│ 대기 시간 > 1ms                         │
│     │                                   │
│     ▼                                   │
│ state |= mutexStarving                  │
│     │                                   │
│     ▼                                   │
│ Unlock 시 다음 waiter에게 직접 전달     │
│ (새 goroutine 경쟁 차단)                │
│     │                                   │
│     ▼                                   │
│ waiter가 락 획득 후:                    │
│ - 마지막 waiter이거나                   │
│ - 대기 시간 < 1ms이면                   │
│ → 일반 모드로 복귀                      │
└─────────────────────────────────────────┘
```

### RWMutex 구조체

```go
// sync/rwmutex.go
type RWMutex struct {
    w           Mutex  // writer 간 상호 배제
    writerSem   uint32 // writer 대기 세마포어
    readerSem   uint32 // reader 대기 세마포어
    readerCount int32  // 현재 reader 수 (음수면 writer 대기)
    readerWait  int32  // writer가 대기 중인 reader 수
}

const rwmutexMaxReaders = 1 << 30  // 최대 reader 수
```

### WaitGroup 구현

```go
// sync/waitgroup.go
type WaitGroup struct {
    noCopy noCopy   // go vet 복사 감지용
    state1 [3]uint32  // counter + waiter + sema
}

// state는 64비트 정렬을 위해 패딩:
// [counter(32bit) + waiter(32bit)] + [sema(32bit)]
// 또는
// [sema(32bit)] + [counter(32bit) + waiter(32bit)]
// (정렬에 따라 다름)

func (wg *WaitGroup) Add(delta int) {
    // atomic으로 counter 증가
    // counter가 0이 되면 모든 waiter 깨우기
}

func (wg *WaitGroup) Wait() {
    // waiter count 증가
    // semacquire로 대기
}

func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

### sync.Once 구현

```go
// sync/once.go
type Once struct {
    done uint32  // 실행 완료 플래그
    m    Mutex   // 초기화 보호
}

func (o *Once) Do(f func()) {
    // Fast path: 이미 완료됨
    if atomic.LoadUint32(&o.done) != 0 {
        return
    }

    // Slow path: 락으로 동기화
    o.m.Lock()
    defer o.m.Unlock()

    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()  // 단 한 번만 실행
    }
}
```

### sync.Map 내부 구조

```go
// sync/map.go
type Map struct {
    mu     Mutex
    read   atomic.Value // readOnly 구조체
    dirty  map[interface{}]*entry
    misses int
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool  // dirty에 read에 없는 키 있음
}

// 읽기 최적화:
// 1. read map 먼저 확인 (락 없이)
// 2. 없으면 dirty map 확인 (락 필요)
// 3. misses가 임계값 넘으면 dirty → read 승격
```

## 탐색

- [Go 개요로 돌아가기](./README.md)
- 이전: [Select 문](./03-select.md)
- 다음: [Context 패키지](./05-context.md)
