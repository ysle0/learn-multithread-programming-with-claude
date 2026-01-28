# Compare-And-Swap (CAS) 연산

## 목차
1. [개요](#개요)
2. [CAS의 원리](#cas의-원리)
3. [하드웨어 지원](#하드웨어-지원)
4. [언어별 CAS 구현](#언어별-cas-구현)
5. [CAS 패턴](#cas-패턴)
6. [CAS의 한계](#cas의-한계)
7. [실전 예제](#실전-예제)
8. [요약](#요약)

---

## 개요

**Compare-And-Swap (CAS)**는 Lock-Free 프로그래밍의 기본 빌딩 블록입니다. 락 없이 원자적으로 값을 비교하고 교환하는 연산으로, 현대 CPU에서 하드웨어 수준으로 지원됩니다.

### CAS의 의미

```
CAS(address, expected, desired):
    atomically {
        if (*address == expected) {
            *address = desired
            return true   // 성공
        } else {
            return false  // 실패 (다른 스레드가 먼저 수정함)
        }
    }
```

### 왜 CAS가 필요한가?

```
┌─────────────────────────────────────────────────────────────────┐
│                    전통적인 락 기반 접근                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread A              Thread B              Thread C           │
│     │                     │                     │               │
│     ▼                     ▼                     ▼               │
│  lock()               lock()                lock()              │
│     │                  (blocked)            (blocked)           │
│     ▼                     │                     │               │
│  counter++                │                     │               │
│     │                     │                     │               │
│     ▼                     │                     │               │
│  unlock() ───────────────►│                     │               │
│                           ▼                     │               │
│                       counter++                 │               │
│                           │                     │               │
│                           ▼                     │               │
│                       unlock() ────────────────►│               │
│                                                 ▼               │
│                                             counter++           │
│                                                                 │
│  문제점: 블로킹, 컨텍스트 스위칭, 우선순위 역전                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    CAS 기반 Lock-Free 접근                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread A              Thread B              Thread C           │
│     │                     │                     │               │
│     ▼                     ▼                     ▼               │
│  read counter=0       read counter=0       read counter=0       │
│     │                     │                     │               │
│     ▼                     ▼                     ▼               │
│  CAS(0→1) ✓           CAS(0→1) ✗           CAS(0→1) ✗          │
│  (성공!)              (실패, 재시도)        (실패, 재시도)        │
│                           │                     │               │
│                           ▼                     ▼               │
│                       read counter=1       read counter=1       │
│                           │                     │               │
│                           ▼                     ▼               │
│                       CAS(1→2) ✓           CAS(1→2) ✗          │
│                       (성공!)              (실패, 재시도)        │
│                                                 │               │
│                                                 ▼               │
│                                             read counter=2      │
│                                                 │               │
│                                                 ▼               │
│                                             CAS(2→3) ✓         │
│                                             (성공!)             │
│                                                                 │
│  장점: 블로킹 없음, 항상 진행 보장 (전체 시스템 기준)              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## CAS의 원리

### 기본 동작 흐름

```c
// CAS 의사 코드 (실제로는 원자적으로 실행됨)
bool compare_and_swap(int* addr, int expected, int desired) {
    // 이 전체 블록이 원자적으로 실행됨
    if (*addr == expected) {
        *addr = desired;
        return true;
    }
    return false;
}
```

### CAS 루프 패턴

CAS는 실패할 수 있으므로 보통 루프와 함께 사용됩니다.

```c
// 원자적 증가 (락 없이)
void atomic_increment(atomic_int* counter) {
    int old_value, new_value;
    do {
        old_value = atomic_load(counter);        // 1. 현재 값 읽기
        new_value = old_value + 1;               // 2. 새 값 계산
    } while (!atomic_compare_exchange_weak(      // 3. CAS 시도
        counter, &old_value, new_value));
    // 실패하면 old_value가 실제 현재 값으로 업데이트됨
}
```

### Strong vs Weak CAS

```c
// Strong CAS: 값이 같으면 반드시 성공
// - 실패 = 다른 스레드가 값을 변경함
bool atomic_compare_exchange_strong(atomic_int* obj, int* expected, int desired);

// Weak CAS: 값이 같아도 실패할 수 있음 (spurious failure)
// - 루프에서 사용 시 더 효율적 (일부 아키텍처에서)
bool atomic_compare_exchange_weak(atomic_int* obj, int* expected, int desired);
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Strong vs Weak CAS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Strong CAS:                                                    │
│  ┌─────────────────────────────────────────────┐               │
│  │ if (값 일치)  → 반드시 성공                   │               │
│  │ if (값 불일치) → 실패                         │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  Weak CAS:                                                      │
│  ┌─────────────────────────────────────────────┐               │
│  │ if (값 일치)  → 성공 또는 spurious failure    │               │
│  │ if (값 불일치) → 실패                         │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  사용 가이드:                                                    │
│  ┌─────────────────────────────────────────────┐               │
│  │ 루프 안에서 → weak (더 효율적)                │               │
│  │ 단일 시도   → strong (명확한 결과)            │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 하드웨어 지원

### x86/x64: CMPXCHG 명령어

```nasm
; x86 CMPXCHG 명령어
; 비교 대상: EAX 레지스터
; 교환 대상: 메모리 위치와 소스 레지스터

; CAS(memory, expected, desired)
; EAX = expected
; EBX = desired
; [addr] = memory location

lock cmpxchg [addr], ebx
; if ([addr] == EAX) {
;     [addr] = EBX
;     ZF = 1 (성공)
; } else {
;     EAX = [addr]  ; expected가 실제 값으로 업데이트됨
;     ZF = 0 (실패)
; }
```

```
┌─────────────────────────────────────────────────────────────────┐
│                x86 LOCK CMPXCHG 동작                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before:                        After (Success):                │
│  ┌─────────┐ ┌─────────┐       ┌─────────┐ ┌─────────┐        │
│  │ EAX: 5  │ │ EBX: 6  │       │ EAX: 5  │ │ EBX: 6  │        │
│  └─────────┘ └─────────┘       └─────────┘ └─────────┘        │
│  ┌─────────┐                   ┌─────────┐                     │
│  │[addr]: 5│                   │[addr]: 6│  ← 교환됨           │
│  └─────────┘                   └─────────┘                     │
│                                ZF = 1 (성공)                    │
│                                                                 │
│  Before:                        After (Failure):                │
│  ┌─────────┐ ┌─────────┐       ┌─────────┐ ┌─────────┐        │
│  │ EAX: 5  │ │ EBX: 6  │       │ EAX: 7  │ │ EBX: 6  │        │
│  └─────────┘ └─────────┘       └─────────┘ └─────────┘        │
│  ┌─────────┐                   ┌─────────┐  ↑                  │
│  │[addr]: 7│  ← 예상과 다름    │[addr]: 7│  실제 값으로 업데이트 │
│  └─────────┘                   └─────────┘                     │
│                                ZF = 0 (실패)                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### x86 Double-Width CAS (CMPXCHG8B/CMPXCHG16B)

```nasm
; 64비트 CAS (32비트 모드)
lock cmpxchg8b [addr]
; EDX:EAX = expected (64-bit)
; ECX:EBX = desired (64-bit)

; 128비트 CAS (64비트 모드)
lock cmpxchg16b [addr]
; RDX:RAX = expected (128-bit)
; RCX:RBX = desired (128-bit)
```

### ARM: LL/SC (Load-Linked / Store-Conditional)

ARM은 CAS 대신 LL/SC 쌍을 사용합니다.

```nasm
; ARM LL/SC 예제 (ARMv8)
retry:
    ldxr  w0, [x1]         ; Load-Exclusive: 값을 읽고 "예약"
    cmp   w0, w2           ; expected와 비교
    b.ne  fail             ; 다르면 실패
    stxr  w3, w4, [x1]     ; Store-Exclusive: 조건부 저장
    cbnz  w3, retry        ; 저장 실패시 재시도
    ; 성공
fail:
    ; 실패
```

```
┌─────────────────────────────────────────────────────────────────┐
│              CAS vs LL/SC 비교                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CAS (x86):                                                     │
│  ┌─────────────────────────────────────────────┐               │
│  │ 단일 명령어로 비교+교환                       │               │
│  │ + 간단함                                     │               │
│  │ - ABA 문제에 취약                            │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  LL/SC (ARM, RISC-V, PowerPC):                                  │
│  ┌─────────────────────────────────────────────┐               │
│  │ Load-Linked: 값 읽기 + 모니터 설정            │               │
│  │ Store-Conditional: 모니터 유효하면 저장       │               │
│  │ + ABA 문제 면역 (값이 아닌 접근을 감지)       │               │
│  │ - Spurious failure 가능                      │               │
│  │ - 두 명령어 사이에 다른 메모리 접근 불가       │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
│  LL/SC의 ABA 면역:                                               │
│  ┌─────────────────────────────────────────────┐               │
│  │ T1: LDXR (값=A)                              │               │
│  │ T2: 값을 A→B→A로 변경                        │               │
│  │ T1: STXR 실패! (값은 A지만 다른 쓰기가 있었음) │               │
│  └─────────────────────────────────────────────┘               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 언어별 CAS 구현

### C11 (stdatomic.h)

```c
#include <stdatomic.h>
#include <stdbool.h>

atomic_int counter = 0;

void increment() {
    int expected = atomic_load(&counter);
    while (!atomic_compare_exchange_weak(&counter, &expected, expected + 1)) {
        // expected는 자동으로 실제 값으로 업데이트됨
    }
}

// 메모리 순서 지정 버전
void increment_explicit() {
    int expected = atomic_load_explicit(&counter, memory_order_relaxed);
    while (!atomic_compare_exchange_weak_explicit(
        &counter, &expected, expected + 1,
        memory_order_release,   // 성공 시 메모리 순서
        memory_order_relaxed    // 실패 시 메모리 순서
    )) {
        // 재시도
    }
}
```

### C++11 (std::atomic)

```cpp
#include <atomic>

std::atomic<int> counter{0};

void increment() {
    int expected = counter.load();
    while (!counter.compare_exchange_weak(expected, expected + 1)) {
        // expected는 자동으로 실제 값으로 업데이트됨
    }
}

// 더 간단한 방법 (내부적으로 CAS 사용)
void increment_simple() {
    counter.fetch_add(1, std::memory_order_relaxed);
}

// 포인터 CAS
std::atomic<Node*> head{nullptr};

void push(Node* new_node) {
    Node* expected = head.load();
    do {
        new_node->next = expected;
    } while (!head.compare_exchange_weak(expected, new_node));
}
```

### Go (sync/atomic)

```go
import "sync/atomic"

var counter int64

func increment() {
    for {
        old := atomic.LoadInt64(&counter)
        if atomic.CompareAndSwapInt64(&counter, old, old+1) {
            break
        }
    }
}

// 더 간단한 방법
func incrementSimple() {
    atomic.AddInt64(&counter, 1)
}

// 포인터 CAS
import "unsafe"

type Node struct {
    value int
    next  unsafe.Pointer
}

var head unsafe.Pointer

func push(node *Node) {
    for {
        oldHead := atomic.LoadPointer(&head)
        node.next = oldHead
        if atomic.CompareAndSwapPointer(&head, oldHead, unsafe.Pointer(node)) {
            break
        }
    }
}
```

### Rust (std::sync::atomic)

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static COUNTER: AtomicI32 = AtomicI32::new(0);

fn increment() {
    let mut expected = COUNTER.load(Ordering::Relaxed);
    loop {
        match COUNTER.compare_exchange_weak(
            expected,
            expected + 1,
            Ordering::Release,
            Ordering::Relaxed,
        ) {
            Ok(_) => break,
            Err(actual) => expected = actual,
        }
    }
}

// 더 간단한 방법
fn increment_simple() {
    COUNTER.fetch_add(1, Ordering::Relaxed);
}
```

### Java (java.util.concurrent.atomic)

```java
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicReference;

AtomicInteger counter = new AtomicInteger(0);

void increment() {
    int expected;
    do {
        expected = counter.get();
    } while (!counter.compareAndSet(expected, expected + 1));
}

// 더 간단한 방법
void incrementSimple() {
    counter.incrementAndGet();
}

// 포인터 CAS
AtomicReference<Node> head = new AtomicReference<>(null);

void push(Node newNode) {
    Node expected;
    do {
        expected = head.get();
        newNode.next = expected;
    } while (!head.compareAndSet(expected, newNode));
}
```

---

## CAS 패턴

### 1. Lock-Free Counter

```c
#include <stdatomic.h>

typedef struct {
    atomic_long value;
} LockFreeCounter;

void counter_init(LockFreeCounter* c) {
    atomic_store(&c->value, 0);
}

long counter_increment(LockFreeCounter* c) {
    return atomic_fetch_add(&c->value, 1) + 1;
}

long counter_decrement(LockFreeCounter* c) {
    return atomic_fetch_sub(&c->value, 1) - 1;
}

long counter_get(LockFreeCounter* c) {
    return atomic_load(&c->value);
}

// CAS를 직접 사용한 조건부 증가
bool counter_increment_if_less_than(LockFreeCounter* c, long max) {
    long expected = atomic_load(&c->value);
    while (expected < max) {
        if (atomic_compare_exchange_weak(&c->value, &expected, expected + 1)) {
            return true;
        }
        // expected가 자동으로 업데이트됨
    }
    return false;
}
```

### 2. Lock-Free Stack (Treiber Stack)

```c
#include <stdatomic.h>
#include <stdlib.h>

typedef struct Node {
    void* data;
    struct Node* next;
} Node;

typedef struct {
    _Atomic(Node*) top;
} LockFreeStack;

void stack_init(LockFreeStack* stack) {
    atomic_store(&stack->top, NULL);
}

void stack_push(LockFreeStack* stack, void* data) {
    Node* new_node = (Node*)malloc(sizeof(Node));
    new_node->data = data;

    Node* expected = atomic_load(&stack->top);
    do {
        new_node->next = expected;
    } while (!atomic_compare_exchange_weak(&stack->top, &expected, new_node));
}

void* stack_pop(LockFreeStack* stack) {
    Node* expected = atomic_load(&stack->top);
    while (expected != NULL) {
        Node* next = expected->next;
        if (atomic_compare_exchange_weak(&stack->top, &expected, next)) {
            void* data = expected->data;
            // 주의: 여기서 free(expected)하면 ABA 문제 발생 가능!
            // Hazard Pointer 또는 다른 메모리 회수 기법 필요
            return data;
        }
        // expected가 자동으로 업데이트됨
    }
    return NULL;  // 스택이 비어있음
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    Lock-Free Stack Push                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Initial:          After push(X):                               │
│                                                                 │
│  top ──► [A]       top ──► [X]                                  │
│           │                 │                                   │
│           ▼                 ▼                                   │
│          [B]               [A]                                  │
│           │                 │                                   │
│           ▼                 ▼                                   │
│          NULL              [B]                                  │
│                             │                                   │
│                             ▼                                   │
│                           NULL                                  │
│                                                                 │
│  Push 과정:                                                      │
│  1. new_node = malloc(X)                                        │
│  2. expected = top (= A)                                        │
│  3. new_node->next = expected (X→A)                            │
│  4. CAS(top, A, X) → 성공하면 완료, 실패하면 2번으로              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Lock-Free Flag (One-time Initialization)

```c
#include <stdatomic.h>

typedef struct {
    atomic_int state;  // 0=uninit, 1=initializing, 2=initialized
    void* data;
} OnceFlag;

#define ONCE_UNINIT      0
#define ONCE_INITIALIZING 1
#define ONCE_INITIALIZED  2

void* once_init(OnceFlag* flag, void* (*init_func)(void)) {
    int expected = ONCE_UNINIT;

    // 초기화 시도
    if (atomic_compare_exchange_strong(&flag->state, &expected, ONCE_INITIALIZING)) {
        // 이 스레드가 초기화를 담당
        flag->data = init_func();
        atomic_store(&flag->state, ONCE_INITIALIZED);
        return flag->data;
    }

    // 다른 스레드가 초기화 중이거나 완료됨
    while (atomic_load(&flag->state) != ONCE_INITIALIZED) {
        // 스핀 또는 yield
        // 실제 구현에서는 더 정교한 대기 필요
    }

    return flag->data;
}
```

### 4. Lock-Free Update (Read-Modify-Write)

```c
#include <stdatomic.h>

typedef struct {
    atomic_int value;
} AtomicMax;

// value를 new_value로 업데이트하되, 더 큰 값만 저장
int atomic_update_max(AtomicMax* m, int new_value) {
    int expected = atomic_load(&m->value);
    while (new_value > expected) {
        if (atomic_compare_exchange_weak(&m->value, &expected, new_value)) {
            return new_value;
        }
        // expected가 업데이트됨
    }
    return expected;  // 기존 값이 더 큼
}

// 범위 내에서만 업데이트
bool atomic_update_if_in_range(atomic_int* v, int min, int max, int new_value) {
    int expected = atomic_load(v);
    while (expected >= min && expected <= max) {
        if (atomic_compare_exchange_weak(v, &expected, new_value)) {
            return true;
        }
    }
    return false;
}
```

### 5. Double-CAS 패턴 (포인터 + 카운터)

ABA 문제를 피하기 위해 포인터와 버전 카운터를 함께 CAS합니다.

```c
#include <stdatomic.h>
#include <stdint.h>

// 16바이트 구조체 (x64에서 CMPXCHG16B 사용)
typedef struct {
    void* ptr;
    uint64_t counter;
} PointerWithCounter;

typedef struct {
    _Atomic(PointerWithCounter) head;
} TaggedStack;

void tagged_push(TaggedStack* stack, void* data) {
    PointerWithCounter expected = atomic_load(&stack->head);
    PointerWithCounter desired;

    do {
        desired.ptr = data;
        desired.counter = expected.counter + 1;
        ((Node*)data)->next = expected.ptr;
    } while (!atomic_compare_exchange_weak(&stack->head, &expected, desired));
}

void* tagged_pop(TaggedStack* stack) {
    PointerWithCounter expected = atomic_load(&stack->head);
    PointerWithCounter desired;

    while (expected.ptr != NULL) {
        desired.ptr = ((Node*)expected.ptr)->next;
        desired.counter = expected.counter + 1;

        if (atomic_compare_exchange_weak(&stack->head, &expected, desired)) {
            return expected.ptr;
        }
    }
    return NULL;
}
```

---

## CAS의 한계

### 1. ABA 문제

```
┌─────────────────────────────────────────────────────────────────┐
│                       ABA Problem                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread 1                    Thread 2                           │
│     │                           │                               │
│  1. read top = A                │                               │
│     │                           │                               │
│  2. (preempted)                 │                               │
│     │                        3. pop A                           │
│     │                           │                               │
│     │                        4. pop B                           │
│     │                           │                               │
│     │                        5. push A (재사용!)                │
│     │                           │                               │
│  6. CAS(top, A, ?) → 성공!      │                               │
│     하지만 스택 상태가 완전히   │                               │
│     달라졌음!                   │                               │
│                                                                 │
│  시나리오:                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 초기: top → A → B → C                                    │   │
│  │                                                          │   │
│  │ T1: top = A를 읽음, next = B를 읽음                      │   │
│  │                                                          │   │
│  │ T2: A pop, B pop, A push (다른 데이터로)                 │   │
│  │     결과: top → A' → C (B는 다른 곳에서 사용 중)         │   │
│  │                                                          │   │
│  │ T1: CAS(top, A, B) → A'==A이므로 성공!                   │   │
│  │     결과: top → B (하지만 B는 이미 해제되었거나           │   │
│  │            다른 곳에서 사용 중!)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**해결책:**
- Tagged Pointer (포인터 + 버전 카운터)
- Hazard Pointers
- Epoch-based Reclamation
- LL/SC 하드웨어 사용

### 2. Livelock (Contention)

```c
// 높은 경합 시 모든 스레드가 계속 실패할 수 있음
void bad_high_contention() {
    while (true) {
        int expected = atomic_load(&counter);
        // 많은 스레드가 동시에 여기 도달
        if (atomic_compare_exchange_weak(&counter, &expected, expected + 1)) {
            break;
        }
        // 대부분 실패 → 재시도 → 또 실패...
    }
}

// 해결: Exponential Backoff
void good_with_backoff() {
    int backoff = 1;
    while (true) {
        int expected = atomic_load(&counter);
        if (atomic_compare_exchange_weak(&counter, &expected, expected + 1)) {
            break;
        }
        // 실패 시 점진적으로 대기 시간 증가
        for (int i = 0; i < backoff; i++) {
            __builtin_ia32_pause();  // CPU hint for spin-wait
        }
        backoff = (backoff < 1024) ? backoff * 2 : 1024;
    }
}
```

### 3. 단일 워드 한계

```c
// 문제: 두 변수를 동시에 원자적으로 업데이트할 수 없음
atomic_int balance_a;
atomic_int balance_b;

// 이것은 원자적이지 않음!
void transfer(int amount) {
    atomic_fetch_sub(&balance_a, amount);  // 여기서 다른 스레드가 볼 수 있음
    atomic_fetch_add(&balance_b, amount);
}

// 해결: 구조체 + Double-CAS 또는 트랜잭션
typedef struct {
    int balance_a;
    int balance_b;
} Balances;

_Atomic(Balances) balances;

void atomic_transfer(int amount) {
    Balances expected = atomic_load(&balances);
    Balances desired;
    do {
        desired.balance_a = expected.balance_a - amount;
        desired.balance_b = expected.balance_b + amount;
    } while (!atomic_compare_exchange_weak(&balances, &expected, desired));
}
```

---

## 실전 예제

### Lock-Free Reference Counter

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct RefCounted {
    atomic_int ref_count;
    void (*destructor)(void*);
    // 실제 데이터는 이 구조체 뒤에 위치
} RefCounted;

RefCounted* ref_create(size_t data_size, void (*destructor)(void*)) {
    RefCounted* obj = malloc(sizeof(RefCounted) + data_size);
    atomic_store(&obj->ref_count, 1);
    obj->destructor = destructor;
    return obj;
}

void* ref_data(RefCounted* obj) {
    return (void*)(obj + 1);
}

void ref_acquire(RefCounted* obj) {
    atomic_fetch_add(&obj->ref_count, 1);
}

void ref_release(RefCounted* obj) {
    if (atomic_fetch_sub(&obj->ref_count, 1) == 1) {
        // 마지막 참조가 해제됨
        if (obj->destructor) {
            obj->destructor(ref_data(obj));
        }
        free(obj);
    }
}

// 조건부 획득: 이미 0이면 실패
bool ref_try_acquire(RefCounted* obj) {
    int expected = atomic_load(&obj->ref_count);
    while (expected > 0) {
        if (atomic_compare_exchange_weak(&obj->ref_count, &expected, expected + 1)) {
            return true;
        }
    }
    return false;
}
```

### Lock-Free Bounded Counter

```c
#include <stdatomic.h>
#include <stdbool.h>

typedef struct {
    atomic_int value;
    int min;
    int max;
} BoundedCounter;

void bounded_init(BoundedCounter* c, int initial, int min, int max) {
    atomic_store(&c->value, initial);
    c->min = min;
    c->max = max;
}

bool bounded_increment(BoundedCounter* c) {
    int expected = atomic_load(&c->value);
    while (expected < c->max) {
        if (atomic_compare_exchange_weak(&c->value, &expected, expected + 1)) {
            return true;
        }
    }
    return false;  // 최대값 도달
}

bool bounded_decrement(BoundedCounter* c) {
    int expected = atomic_load(&c->value);
    while (expected > c->min) {
        if (atomic_compare_exchange_weak(&c->value, &expected, expected - 1)) {
            return true;
        }
    }
    return false;  // 최소값 도달
}

// Semaphore처럼 사용
typedef BoundedCounter LockFreeSemaphore;

bool semaphore_acquire(LockFreeSemaphore* sem) {
    return bounded_decrement(sem);
}

void semaphore_release(LockFreeSemaphore* sem) {
    bounded_increment(sem);
}
```

### Lock-Free Ticket Lock (공정한 스핀락)

```c
#include <stdatomic.h>

typedef struct {
    atomic_uint next_ticket;
    atomic_uint now_serving;
} TicketLock;

void ticket_init(TicketLock* lock) {
    atomic_store(&lock->next_ticket, 0);
    atomic_store(&lock->now_serving, 0);
}

void ticket_lock(TicketLock* lock) {
    // 티켓 발급 (원자적 증가)
    unsigned my_ticket = atomic_fetch_add(&lock->next_ticket, 1);

    // 내 차례가 될 때까지 대기
    while (atomic_load(&lock->now_serving) != my_ticket) {
        __builtin_ia32_pause();
    }
}

void ticket_unlock(TicketLock* lock) {
    // 다음 티켓 호출
    atomic_fetch_add(&lock->now_serving, 1);
}
```

---

## 요약

### CAS 핵심 정리

| 항목 | 설명 |
|------|------|
| **정의** | 원자적으로 값 비교 후 일치하면 교환 |
| **성공 조건** | 메모리 값 == expected |
| **실패 시** | expected가 실제 값으로 업데이트됨 |
| **하드웨어** | x86: CMPXCHG, ARM: LL/SC |

### 사용 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│                    CAS 사용 결정 트리                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  단순 증가/감소?                                                 │
│      │                                                          │
│      ├── Yes → fetch_add/fetch_sub 사용                        │
│      │                                                          │
│      └── No → 조건부 업데이트?                                  │
│               │                                                 │
│               ├── Yes → CAS 루프 사용                          │
│               │    └── 루프 안 → weak CAS                      │
│               │    └── 단일 시도 → strong CAS                   │
│               │                                                 │
│               └── No → 복잡한 연산?                             │
│                    │                                            │
│                    ├── Yes → 구조체 + Double-CAS               │
│                    │         또는 락 사용 고려                  │
│                    │                                            │
│                    └── No → 단순 load/store                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 주의사항

1. **ABA 문제**: 포인터 CAS 시 Tagged Pointer 또는 메모리 회수 기법 필요
2. **Contention**: 높은 경합 시 Backoff 전략 적용
3. **Memory Ordering**: CAS의 메모리 순서 이해 필수 (다음 문서에서 상세 설명)
4. **단일 워드 한계**: 여러 변수 동시 업데이트 시 구조체화 필요

---

## 관련 문서

- [Memory Ordering](./02-memory-ordering.md) - CAS와 메모리 순서
- [Lock-Free Stack](./03-lock-free-stack.md) - CAS 기반 스택 구현
- [Lock-Free Queue](./04-lock-free-queue.md) - CAS 기반 큐 구현
- [ABA Problem](./07-aba-problem.md) - ABA 문제와 해결책
- [Hazard Pointers](./08-hazard-pointers.md) - 안전한 메모리 회수

---

## 참고 자료

- [C++ Concurrency in Action, Chapter 5](https://www.manning.com/books/c-plus-plus-concurrency-in-action)
- [Intel® 64 and IA-32 Architectures Software Developer's Manual](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)
- [Preshing on Programming - Compare-And-Swap](https://preshing.com/20150402/you-can-do-any-kind-of-atomic-read-modify-write-operation/)
- [1024cores - Lock-Free Algorithms](http://www.1024cores.net/home/lock-free-algorithms)
