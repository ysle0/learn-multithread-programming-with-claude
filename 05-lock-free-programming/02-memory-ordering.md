# Memory Ordering (메모리 순서)

## 목차
1. [개요](#개요)
2. [왜 메모리 순서가 중요한가?](#왜-메모리-순서가-중요한가)
3. [컴파일러와 CPU의 재배열](#컴파일러와-cpu의-재배열)
4. [메모리 순서 모델](#메모리-순서-모델)
5. [언어별 Memory Ordering](#언어별-memory-ordering)
6. [실전 패턴](#실전-패턴)
7. [하드웨어 관점](#하드웨어-관점)
8. [요약](#요약)

---

## 개요

**Memory Ordering**은 멀티스레드 프로그램에서 메모리 연산의 실행 순서를 제어하는 메커니즘입니다. 락 없이 스레드 간 통신을 할 때 필수적인 개념입니다.

### 핵심 문제

```c
// Thread 1                    // Thread 2
data = 42;                     while (!ready);  // ready가 true가 될 때까지 대기
ready = true;                  print(data);     // 42가 출력될까?
```

**직관적 예상**: data=42가 먼저 쓰이고, ready=true가 나중에 쓰이므로, Thread 2는 42를 출력할 것이다.

**현실**: CPU나 컴파일러가 순서를 바꿀 수 있어서 **0이 출력될 수 있다!**

---

## 왜 메모리 순서가 중요한가?

### 문제 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│            메모리 재배열로 인한 버그                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  원래 코드:                     실제 실행 (재배열 후):           │
│                                                                 │
│  Thread 1:                     Thread 1:                        │
│  ┌─────────────────┐           ┌─────────────────┐              │
│  │ 1. data = 42    │           │ 1. ready = true │ ← 순서 바뀜  │
│  │ 2. ready = true │           │ 2. data = 42    │              │
│  └─────────────────┘           └─────────────────┘              │
│                                                                 │
│  Thread 2:                     Thread 2:                        │
│  ┌─────────────────┐           ┌─────────────────┐              │
│  │ while(!ready);  │           │ while(!ready);  │              │
│  │ print(data);    │           │ print(data);    │ → 0 출력!    │
│  └─────────────────┘           └─────────────────┘              │
│                                                                 │
│  타임라인:                                                       │
│  ────────────────────────────────────────────────────────────   │
│  T1: ───────[ready=true]───────────────[data=42]───────────►   │
│  T2: ─────────────────────[see ready]──[read data=0]────────►  │
│                             ↑                                   │
│                    ready는 보지만 data는 아직 0                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 재배열이 발생하는 이유

1. **컴파일러 최적화**
   - 불필요한 메모리 접근 제거
   - 명령어 순서 변경으로 파이프라인 효율화
   - 레지스터 할당 최적화

2. **CPU Out-of-Order 실행**
   - 독립적인 명령어를 병렬 실행
   - 캐시 미스 시 다른 명령어 먼저 실행
   - Store Buffer로 인한 지연

3. **캐시 일관성 지연**
   - 다른 CPU가 쓴 값이 즉시 보이지 않음
   - 캐시 계층 간 전파 시간

---

## 컴파일러와 CPU의 재배열

### 컴파일러 재배열

```c
// 원본 코드
int a = 1;
int b = 2;
int c = 3;

// 컴파일러가 생성할 수 있는 코드 (최적화)
int c = 3;  // 순서 변경!
int a = 1;
int b = 2;

// 또는
int b = 2;
int c = 3;
int a = 1;
```

```c
// 컴파일러 재배열 방지
int a = 1;
__asm__ __volatile__("" ::: "memory");  // 컴파일러 배리어
int b = 2;

// C11 방식
atomic_signal_fence(memory_order_seq_cst);
```

### CPU 재배열 종류

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU 재배열 유형                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Store-Store 재배열:                                          │
│     원본: Store A, Store B                                      │
│     실제: Store B, Store A (또는 동시에)                         │
│                                                                 │
│  2. Load-Load 재배열:                                            │
│     원본: Load A, Load B                                        │
│     실제: Load B, Load A                                        │
│                                                                 │
│  3. Load-Store 재배열:                                           │
│     원본: Load A, Store B                                       │
│     실제: Store B, Load A                                       │
│                                                                 │
│  4. Store-Load 재배열: (가장 흔함)                               │
│     원본: Store A, Load B                                       │
│     실제: Load B, Store A                                       │
│                                                                 │
│  플랫폼별 허용 재배열:                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 아키텍처  │ S-S │ L-L │ L-S │ S-L │                     │   │
│  │──────────│─────│─────│─────│─────│                     │   │
│  │ x86/x64  │  ✗  │  ✗  │  ✗  │  ✓  │ (강한 모델)         │   │
│  │ ARM      │  ✓  │  ✓  │  ✓  │  ✓  │ (약한 모델)         │   │
│  │ PowerPC  │  ✓  │  ✓  │  ✓  │  ✓  │ (약한 모델)         │   │
│  │ RISC-V   │  ✓  │  ✓  │  ✓  │  ✓  │ (약한 모델)         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ✓ = 재배열 가능, ✗ = 재배열 불가                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Store Buffer 문제

```
┌─────────────────────────────────────────────────────────────────┐
│                    Store Buffer                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU 0                              CPU 1                       │
│  ┌─────────────┐                   ┌─────────────┐             │
│  │    Core     │                   │    Core     │             │
│  └──────┬──────┘                   └──────┬──────┘             │
│         │                                 │                     │
│  ┌──────▼──────┐                   ┌──────▼──────┐             │
│  │Store Buffer │                   │Store Buffer │             │
│  │ [x=1 대기중]│                   │ [y=1 대기중]│             │
│  └──────┬──────┘                   └──────┬──────┘             │
│         │                                 │                     │
│  ┌──────▼──────┐                   ┌──────▼──────┐             │
│  │   L1 Cache  │                   │   L1 Cache  │             │
│  │   x=0, y=0  │                   │   x=0, y=0  │             │
│  └──────┬──────┘                   └──────┬──────┘             │
│         │                                 │                     │
│         └────────────┬────────────────────┘                    │
│                      │                                          │
│               ┌──────▼──────┐                                   │
│               │  L3 Cache / │                                   │
│               │   Memory    │                                   │
│               └─────────────┘                                   │
│                                                                 │
│  문제 시나리오:                                                  │
│  CPU 0: x = 1; r1 = y;  // x=1은 Store Buffer에, y는 캐시에서   │
│  CPU 1: y = 1; r2 = x;  // y=1은 Store Buffer에, x는 캐시에서   │
│                                                                 │
│  결과: r1 = 0, r2 = 0 가능! (둘 다 상대방의 Store를 못 봄)       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 메모리 순서 모델

### C++11/C11 Memory Order

```cpp
enum memory_order {
    memory_order_relaxed,    // 가장 약함: 순서 보장 없음
    memory_order_consume,    // 데이터 의존성만 보장 (거의 사용 안함)
    memory_order_acquire,    // 이 Load 이후의 연산이 앞으로 오지 않음
    memory_order_release,    // 이 Store 이전의 연산이 뒤로 가지 않음
    memory_order_acq_rel,    // acquire + release
    memory_order_seq_cst     // 가장 강함: 전역 순서 보장
};
```

### 각 순서의 의미

#### 1. Relaxed (memory_order_relaxed)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Relaxed Ordering                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  보장: 원자성만 (값이 찢어지지 않음)                              │
│  비보장: 다른 스레드에서 보이는 순서                              │
│                                                                 │
│  Thread 1:                          Thread 2:                   │
│  x.store(1, relaxed);              r1 = y.load(relaxed);        │
│  y.store(1, relaxed);              r2 = x.load(relaxed);        │
│                                                                 │
│  가능한 결과: r1=1, r2=0 (y는 봤지만 x는 못 봄)                  │
│                                                                 │
│  사용처: 단순 카운터, 통계 수집 (순서 무관한 경우)                │
│                                                                 │
│  예시:                                                           │
│  atomic<int> counter{0};                                        │
│  counter.fetch_add(1, memory_order_relaxed);  // OK             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 2. Acquire-Release

```
┌─────────────────────────────────────────────────────────────────┐
│                 Acquire-Release Ordering                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Release (Store):                                               │
│  ┌─────────────────────────────────────────────┐               │
│  │ 이전의 모든 읽기/쓰기가 이 Store 뒤로 가지 않음│               │
│  │                                              │               │
│  │    ↑ 모든 이전 연산                          │               │
│  │ ───┼────────────────── (배리어)              │               │
│  │    ↓ release store                           │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  Acquire (Load):                                                │
│  ┌─────────────────────────────────────────────┐               │
│  │ 이 Load 이후의 모든 읽기/쓰기가 앞으로 오지 않음│               │
│  │                                              │               │
│  │    ↑ acquire load                            │               │
│  │ ───┼────────────────── (배리어)              │               │
│  │    ↓ 모든 이후 연산                          │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  동기화 쌍:                                                      │
│  ┌─────────────────────────────────────────────┐               │
│  │ Thread 1 (Producer)    Thread 2 (Consumer)  │               │
│  │                                              │               │
│  │ data = 42;             ───────────────────► │               │
│  │ flag.store(1, release);  if (flag.load(     │               │
│  │         │                    acquire) == 1) │               │
│  │         │                        │          │               │
│  │         └─────── sync ──────────►│          │               │
│  │                                   │          │               │
│  │                              use(data);     │               │
│  │                              // 42 보장!    │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```cpp
// Acquire-Release 예제
std::atomic<bool> ready{false};
int data = 0;

// Producer
void producer() {
    data = 42;                                    // (1) 일반 쓰기
    ready.store(true, std::memory_order_release); // (2) release store
    // (1)은 절대로 (2) 뒤로 가지 않음
}

// Consumer
void consumer() {
    while (!ready.load(std::memory_order_acquire)); // (3) acquire load
    assert(data == 42);                              // (4) 일반 읽기
    // (4)는 절대로 (3) 앞으로 오지 않음
    // (2)와 (3)이 동기화되어 (1)의 결과가 (4)에서 보임
}
```

#### 3. Sequential Consistency (seq_cst)

```
┌─────────────────────────────────────────────────────────────────┐
│              Sequential Consistency                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  보장: 모든 스레드가 동일한 전역 순서를 관찰                      │
│                                                                 │
│  Thread 1:              Thread 2:              관찰 순서:       │
│  x.store(1);           y.store(1);            모든 스레드가     │
│                                                동일하게 봄      │
│  가능한 전역 순서들:                                            │
│  1) x=1 → y=1                                                  │
│  2) y=1 → x=1                                                  │
│  (하나가 선택되면 모두 같은 순서를 봄)                           │
│                                                                 │
│  예제: Dekker's Algorithm                                       │
│  ┌─────────────────────────────────────────────┐               │
│  │ atomic<bool> x{false}, y{false};            │               │
│  │ atomic<int> z{0};                            │               │
│  │                                              │               │
│  │ Thread 1:              Thread 2:            │               │
│  │ x.store(true);         y.store(true);       │               │
│  │ if (!y.load())         if (!x.load())       │               │
│  │   z++;                   z++;               │               │
│  │                                              │               │
│  │ seq_cst: z는 최대 1 (둘 다 동시 진입 불가)  │               │
│  │ relaxed: z가 2가 될 수 있음! (위험)         │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  비용: 가장 느림 (전체 메모리 배리어)                            │
│  사용: 정확성이 성능보다 중요할 때, 기본값                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Memory Order 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│              Memory Order 선택 플로우차트                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  스레드 간 데이터 전달이 필요한가?                               │
│         │                                                       │
│         ├── No ──► relaxed                                     │
│         │         (카운터, 통계)                                 │
│         │                                                       │
│         └── Yes ──► 단방향 통신인가?                            │
│                      │                                          │
│                      ├── Yes ──► acquire/release 쌍            │
│                      │          (Producer-Consumer)             │
│                      │                                          │
│                      └── No ──► 여러 변수의 전역 순서가         │
│                                 필요한가?                        │
│                                  │                              │
│                                  ├── No ──► acq_rel            │
│                                  │         (양방향 동기화)      │
│                                  │                              │
│                                  └── Yes ──► seq_cst           │
│                                             (가장 안전)         │
│                                                                 │
│  성능 순위: relaxed > acquire/release > acq_rel > seq_cst      │
│  안전 순위: seq_cst > acq_rel > acquire/release > relaxed      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 언어별 Memory Ordering

### C11

```c
#include <stdatomic.h>

atomic_int counter = 0;
atomic_bool flag = false;
int data = 0;

// Relaxed
void count() {
    atomic_fetch_add_explicit(&counter, 1, memory_order_relaxed);
}

// Acquire-Release
void producer() {
    data = 42;
    atomic_store_explicit(&flag, true, memory_order_release);
}

void consumer() {
    while (!atomic_load_explicit(&flag, memory_order_acquire));
    assert(data == 42);
}

// Sequential Consistency (기본값)
void default_order() {
    atomic_store(&flag, true);  // seq_cst
    bool b = atomic_load(&flag); // seq_cst
}
```

### C++11

```cpp
#include <atomic>

std::atomic<int> counter{0};
std::atomic<bool> flag{false};
int data = 0;

// Relaxed
void count() {
    counter.fetch_add(1, std::memory_order_relaxed);
}

// Acquire-Release
void producer() {
    data = 42;
    flag.store(true, std::memory_order_release);
}

void consumer() {
    while (!flag.load(std::memory_order_acquire));
    assert(data == 42);
}

// RMW 연산의 memory order
void rmw_example() {
    // exchange는 read와 write 모두 수행
    int old = counter.exchange(10, std::memory_order_acq_rel);

    // compare_exchange는 성공/실패 시 다른 순서 지정 가능
    int expected = 5;
    counter.compare_exchange_strong(
        expected, 10,
        std::memory_order_release,   // 성공 시
        std::memory_order_relaxed    // 실패 시
    );
}
```

### Go

```go
import "sync/atomic"

var counter int64
var flag int32
var data int

// Go는 명시적 memory order가 없음
// atomic 연산은 sequential consistency 제공

func producer() {
    data = 42
    atomic.StoreInt32(&flag, 1)  // release semantics
}

func consumer() {
    for atomic.LoadInt32(&flag) != 1 {}  // acquire semantics
    // data == 42 보장
}

// Go 1.19+: atomic.Int64 타입 제공
import "sync/atomic"

var counter atomic.Int64

func count() {
    counter.Add(1)
}
```

### Rust

```rust
use std::sync::atomic::{AtomicBool, AtomicI32, Ordering};

static COUNTER: AtomicI32 = AtomicI32::new(0);
static FLAG: AtomicBool = AtomicBool::new(false);
static mut DATA: i32 = 0;

// Relaxed
fn count() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
}

// Acquire-Release
fn producer() {
    unsafe { DATA = 42; }
    FLAG.store(true, Ordering::Release);
}

fn consumer() {
    while !FLAG.load(Ordering::Acquire) {}
    unsafe { assert_eq!(DATA, 42); }
}

// SeqCst (기본 권장)
fn safe_default() {
    FLAG.store(true, Ordering::SeqCst);
    let b = FLAG.load(Ordering::SeqCst);
}
```

### Java (VarHandle, Java 9+)

```java
import java.lang.invoke.MethodHandles;
import java.lang.invoke.VarHandle;

public class MemoryOrderExample {
    private volatile int flag = 0;
    private int data = 0;

    // volatile은 acquire-release semantics 제공
    void producer() {
        data = 42;
        flag = 1;  // release
    }

    void consumer() {
        while (flag != 1);  // acquire
        assert data == 42;
    }

    // VarHandle로 더 세밀한 제어 (Java 9+)
    private static final VarHandle FLAG_HANDLE;
    static {
        try {
            FLAG_HANDLE = MethodHandles.lookup()
                .findVarHandle(MemoryOrderExample.class, "flag", int.class);
        } catch (Exception e) {
            throw new Error(e);
        }
    }

    void relaxedIncrement() {
        FLAG_HANDLE.getAndAddAcquire(this, 1);
    }

    void opaqueRead() {
        int v = (int) FLAG_HANDLE.getOpaque(this);  // relaxed-like
    }
}
```

---

## 실전 패턴

### 1. Double-Checked Locking

```cpp
#include <atomic>
#include <mutex>

class Singleton {
    static std::atomic<Singleton*> instance;
    static std::mutex mutex;

public:
    static Singleton* getInstance() {
        Singleton* p = instance.load(std::memory_order_acquire);
        if (p == nullptr) {
            std::lock_guard<std::mutex> lock(mutex);
            p = instance.load(std::memory_order_relaxed);
            if (p == nullptr) {
                p = new Singleton();
                instance.store(p, std::memory_order_release);
            }
        }
        return p;
    }
};

std::atomic<Singleton*> Singleton::instance{nullptr};
std::mutex Singleton::mutex;
```

### 2. Seqlock (Reader-Writer 최적화)

```cpp
#include <atomic>

class Seqlock {
    std::atomic<unsigned> seq{0};
    int data1 = 0;
    int data2 = 0;

public:
    void write(int d1, int d2) {
        unsigned s = seq.load(std::memory_order_relaxed);
        seq.store(s + 1, std::memory_order_relaxed);  // 홀수 = 쓰기 중
        std::atomic_thread_fence(std::memory_order_release);

        data1 = d1;
        data2 = d2;

        std::atomic_thread_fence(std::memory_order_release);
        seq.store(s + 2, std::memory_order_relaxed);  // 짝수 = 쓰기 완료
    }

    bool read(int& d1, int& d2) {
        unsigned s1 = seq.load(std::memory_order_acquire);
        if (s1 & 1) return false;  // 쓰기 중

        d1 = data1;
        d2 = data2;

        std::atomic_thread_fence(std::memory_order_acquire);
        unsigned s2 = seq.load(std::memory_order_relaxed);

        return s1 == s2;  // 읽는 동안 변경되지 않았으면 성공
    }
};
```

### 3. SPSC Queue (Single Producer Single Consumer)

```cpp
#include <atomic>
#include <array>

template<typename T, size_t Size>
class SPSCQueue {
    std::array<T, Size> buffer;
    std::atomic<size_t> head{0};  // Consumer가 읽는 위치
    std::atomic<size_t> tail{0};  // Producer가 쓰는 위치

public:
    bool push(const T& item) {
        size_t t = tail.load(std::memory_order_relaxed);
        size_t next = (t + 1) % Size;

        if (next == head.load(std::memory_order_acquire)) {
            return false;  // Full
        }

        buffer[t] = item;
        tail.store(next, std::memory_order_release);
        return true;
    }

    bool pop(T& item) {
        size_t h = head.load(std::memory_order_relaxed);

        if (h == tail.load(std::memory_order_acquire)) {
            return false;  // Empty
        }

        item = buffer[h];
        head.store((h + 1) % Size, std::memory_order_release);
        return true;
    }
};
```

### 4. Reference Counter

```cpp
#include <atomic>

class RefCounted {
    mutable std::atomic<int> ref_count{1};

public:
    void addRef() const {
        ref_count.fetch_add(1, std::memory_order_relaxed);
        // relaxed OK: ref_count는 순서와 무관
    }

    void release() const {
        if (ref_count.fetch_sub(1, std::memory_order_acq_rel) == 1) {
            // acq_rel: 이전 release들과 동기화 + delete 전 모든 접근 완료 보장
            delete this;
        }
    }
};
```

### 5. Spinlock with Backoff

```cpp
#include <atomic>
#include <thread>

class Spinlock {
    std::atomic<bool> locked{false};

public:
    void lock() {
        for (;;) {
            // 먼저 relaxed로 확인 (최적화)
            if (!locked.exchange(true, std::memory_order_acquire)) {
                return;
            }

            // 락이 잡혀있으면 스핀
            while (locked.load(std::memory_order_relaxed)) {
                // CPU hint for spin-wait
                #if defined(__x86_64__) || defined(_M_X64)
                    __builtin_ia32_pause();
                #elif defined(__aarch64__)
                    asm volatile("yield");
                #endif
            }
        }
    }

    void unlock() {
        locked.store(false, std::memory_order_release);
    }
};
```

---

## 하드웨어 관점

### x86/x64 메모리 모델

```
┌─────────────────────────────────────────────────────────────────┐
│                    x86 Memory Model                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  x86은 "강한" 메모리 모델 (TSO: Total Store Order)              │
│                                                                 │
│  자동 보장:                                                      │
│  - Load → Load (순서 유지)                                       │
│  - Store → Store (순서 유지)                                     │
│  - Load → Store (순서 유지)                                      │
│                                                                 │
│  보장 안됨:                                                      │
│  - Store → Load (재배열 가능!)                                   │
│                                                                 │
│  Memory Fence:                                                   │
│  ┌─────────────────────────────────────────────┐               │
│  │ MFENCE: 모든 이전 읽기/쓰기 완료 보장        │               │
│  │ LFENCE: 모든 이전 읽기 완료 보장             │               │
│  │ SFENCE: 모든 이전 쓰기 완료 보장             │               │
│  │                                              │               │
│  │ LOCK prefix: 암시적 full fence               │               │
│  │ (lock cmpxchg, lock xadd 등)                │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  실제 영향:                                                      │
│  - relaxed와 acquire/release 차이가 작음                        │
│  - seq_cst만 MFENCE 필요 (Store에서)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### ARM 메모리 모델

```
┌─────────────────────────────────────────────────────────────────┐
│                    ARM Memory Model                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ARM은 "약한" 메모리 모델                                        │
│                                                                 │
│  기본적으로 모든 재배열 가능:                                    │
│  - Load → Load                                                  │
│  - Store → Store                                                │
│  - Load → Store                                                 │
│  - Store → Load                                                 │
│                                                                 │
│  Memory Barrier:                                                 │
│  ┌─────────────────────────────────────────────┐               │
│  │ DMB (Data Memory Barrier):                   │               │
│  │   - DMB ISH: Inner Shareable 도메인 배리어   │               │
│  │   - DMB OSH: Outer Shareable 도메인 배리어   │               │
│  │                                              │               │
│  │ DSB (Data Synchronization Barrier):          │               │
│  │   - DMB + 명령어 완료까지 대기               │               │
│  │                                              │               │
│  │ ISB (Instruction Synchronization Barrier):   │               │
│  │   - 파이프라인 플러시                        │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  ARMv8 Atomic 명령어:                                            │
│  ┌─────────────────────────────────────────────┐               │
│  │ LDAPR: Load-Acquire (RCpc)                   │               │
│  │ LDAR:  Load-Acquire                          │               │
│  │ STLR:  Store-Release                         │               │
│  │ CAS:   Compare-And-Swap (ARMv8.1)            │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  실제 영향:                                                      │
│  - memory order 차이가 실제 성능에 영향                         │
│  - relaxed가 확실히 더 빠름                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 플랫폼별 seq_cst 비용

```
┌─────────────────────────────────────────────────────────────────┐
│              seq_cst Store 구현 비용                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  x86-64:                                                        │
│  ┌─────────────────────────────────────────────┐               │
│  │ ; seq_cst store                              │               │
│  │ mov [addr], value                            │               │
│  │ mfence              ; 또는 xchg 사용         │               │
│  └─────────────────────────────────────────────┘               │
│  비용: ~20-50 cycles (mfence)                                   │
│                                                                 │
│  ARM64:                                                          │
│  ┌─────────────────────────────────────────────┐               │
│  │ ; seq_cst store                              │               │
│  │ stlr x0, [x1]       ; store-release          │               │
│  │ dmb ish             ; full barrier           │               │
│  └─────────────────────────────────────────────┘               │
│  비용: ~20-40 cycles                                            │
│                                                                 │
│  vs release store:                                               │
│  ┌─────────────────────────────────────────────┐               │
│  │ x86: mov [addr], value  ; 그냥 store         │               │
│  │ ARM: stlr x0, [x1]      ; store-release      │               │
│  └─────────────────────────────────────────────┘               │
│  비용: ~1-5 cycles                                              │
│                                                                 │
│  결론: 핫 패스에서는 적절한 memory order 선택이 중요             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 요약

### Memory Order 정리표

| Order | Store | Load | RMW | 사용 사례 |
|-------|-------|------|-----|----------|
| relaxed | 원자적 | 원자적 | 원자적 | 카운터, 통계 |
| acquire | - | ↓차단 | ↓차단 | Consumer 측 동기화 |
| release | ↑차단 | - | ↑차단 | Producer 측 동기화 |
| acq_rel | - | - | ↑↓차단 | 양방향 동기화 |
| seq_cst | 전역순서 | 전역순서 | 전역순서 | 기본, 안전 |

### 선택 가이드

```cpp
// 1. 확실하지 않으면 seq_cst (기본값)
atomic.store(val);
atomic.load();

// 2. Producer-Consumer 패턴
producer: data_ready.store(true, release);
consumer: while(!data_ready.load(acquire));

// 3. 단순 카운터/통계
counter.fetch_add(1, relaxed);

// 4. 참조 카운터
addRef: count.fetch_add(1, relaxed);
release: if (count.fetch_sub(1, acq_rel) == 1) delete;
```

### 자주 하는 실수

1. **relaxed 남용**: 데이터 전달 시 relaxed 사용 → 버그
2. **acquire만 사용**: release 없이 acquire만 → 동기화 안됨
3. **x86에서 테스트**: ARM에서 버그 발생 (x86은 강한 모델)
4. **volatile 혼동**: volatile ≠ atomic (C++에서)

---

## 관련 문서

- [CAS 연산](./01-cas-operation.md) - CAS와 memory order 조합
- [Memory Barrier](../02-synchronization/05-memory-barrier.md) - 배리어 상세
- [Atomic Operations](../02-synchronization/04-atomic-operations.md) - 원자적 연산
- [Lock-Free Stack](./03-lock-free-stack.md) - memory order 적용 예

---

## 참고 자료

- [C++ Memory Model - cppreference](https://en.cppreference.com/w/cpp/atomic/memory_order)
- [Preshing on Programming - Memory Barriers](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
- [ARM Barrier Litmus Tests](https://developer.arm.com/documentation/genc007826/latest)
- [Intel® 64 Architecture Memory Ordering White Paper](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Herb Sutter - atomic<> Weapons](https://herbsutter.com/2013/02/11/atomic-weapons-the-c-memory-model-and-modern-hardware/)
