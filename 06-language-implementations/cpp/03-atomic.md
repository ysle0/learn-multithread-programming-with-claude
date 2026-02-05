# C++의 Atomic 연산

Atomic 연산은 단순한 데이터 타입에 대해 Lock-Free 동기화를 제공합니다. 고성능 동시성 프로그래밍과 Lock-Free 알고리즘을 이해하는 데 필수적입니다.

## 목차
- [기본 개념](#기본-개념)
- [std::atomic 타입](#stdatomic-타입)
- [메모리 순서](#메모리-순서)
- [Atomic 연산](#atomic-연산)
- [Compare-and-Swap](#compare-and-swap)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### Atomic이란?

Atomic 연산은 다른 스레드의 간섭 없이 완전히 수행되는 분리 불가능한 연산입니다:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

// 비원자적 - 경쟁 조건!
int counter = 0;

void bad_increment() {
    for (int i = 0; i < 100000; ++i) {
        ++counter;  // Read-Modify-Write - 원자적이지 않음!
    }
}

// 원자적 - 잠금 없이 안전
std::atomic<int> atomic_counter{0};

void good_increment() {
    for (int i = 0; i < 100000; ++i) {
        ++atomic_counter;  // 원자적 연산
    }
}

int main() {
    std::thread t1(good_increment);
    std::thread t2(good_increment);
    t1.join();
    t2.join();
    std::cout << "Result: " << atomic_counter << "\n";  // 항상 200000
    return 0;
}
```

### Atomic을 사용하는 이유

1. **Lock-Free**: mutex 오버헤드 없음
2. **빠름**: 하드웨어 지원 연산
3. **간단함**: 기본적인 동기화에 적합
4. **기반**: 복잡한 Lock-Free 구조의 빌딩 블록

## std::atomic 타입

### 기본 Atomic 타입

```cpp
#include <atomic>
#include <iostream>

int main() {
    // 정수 타입
    std::atomic<int> atomic_int{0};
    std::atomic<long> atomic_long{0};
    std::atomic<unsigned> atomic_uint{0};

    // 불리언
    std::atomic<bool> atomic_flag{false};

    // 포인터
    int value = 42;
    std::atomic<int*> atomic_ptr{&value};

    // 사용자 정의 타입 (trivially copyable이어야 함)
    struct Point {
        int x, y;
    };
    std::atomic<Point> atomic_point{{0, 0}};

    return 0;
}
```

### Atomic 타입 별칭

```cpp
#include <atomic>

int main() {
    // 편의 타입 별칭
    std::atomic_int ai{0};           // std::atomic<int>과 동일
    std::atomic_long al{0};          // std::atomic<long>과 동일
    std::atomic_bool ab{false};      // std::atomic<bool>과 동일

    // 고정 너비 타입
    std::atomic_int32_t ai32{0};
    std::atomic_int64_t ai64{0};

    return 0;
}
```

### std::atomic_flag

유일하게 Lock-Free가 보장되는 atomic:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // 스핀 대기
        }
    }

    void unlock() {
        flag.clear(std::memory_order_release);
    }
};

Spinlock spinlock;
int counter = 0;

void increment() {
    for (int i = 0; i < 10000; ++i) {
        spinlock.lock();
        ++counter;
        spinlock.unlock();
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n";
    return 0;
}
```

## 메모리 순서

### 메모리 순서 옵션

C++는 메모리 동기화에 대한 세밀한 제어를 제공합니다:

```cpp
namespace std {
    enum memory_order {
        memory_order_relaxed,   // 동기화 없음
        memory_order_consume,   // 데이터 의존성 (거의 사용되지 않음)
        memory_order_acquire,   // Acquire 배리어
        memory_order_release,   // Release 배리어
        memory_order_acq_rel,   // Acquire와 Release 모두
        memory_order_seq_cst    // 순차적 일관성 (기본값)
    };
}
```

### 순차적 일관성 (기본값)

가장 강한 순서 - 모든 스레드에서 연산이 동일한 순서로 나타남:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> x{0}, y{0};

void write_x() {
    x.store(1, std::memory_order_seq_cst);  // 기본값
}

void write_y() {
    y.store(1, std::memory_order_seq_cst);
}

void read_values() {
    int r1 = y.load(std::memory_order_seq_cst);
    int r2 = x.load(std::memory_order_seq_cst);
    // r1 == 1이면, r2도 반드시 1이어야 함 (전체 순서 보장)
}
```

### Relaxed 순서

동기화 없음, 원자성만 보장:

```cpp
#include <atomic>
#include <thread>

std::atomic<int> counter{0};

void increment_relaxed() {
    for (int i = 0; i < 10000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(increment_relaxed);
    std::thread t2(increment_relaxed);
    t1.join();
    t2.join();
    // 카운터는 정확하지만, 순서 보장 없음
    return 0;
}
```

### Acquire-Release 순서

동기화에 가장 일반적으로 사용됨:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

std::atomic<bool> ready{false};
int data = 0;

void producer() {
    data = 42;                                    // 1
    ready.store(true, std::memory_order_release); // 2
    // release 전의 모든 쓰기가 acquire 후에 보임
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // 3
        // 대기
    }
    assert(data == 42);  // data = 42를 볼 수 있음이 보장됨
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```

### 메모리 순서 비교

```cpp
#include <atomic>
#include <thread>

std::atomic<int> value{0};

void examples() {
    // 순차적 일관성 - 가장 강함, 가장 느림
    value.store(1, std::memory_order_seq_cst);
    int v1 = value.load(std::memory_order_seq_cst);

    // Acquire-Release - 균형 잡힘
    value.store(2, std::memory_order_release);
    int v2 = value.load(std::memory_order_acquire);

    // Relaxed - 가장 약함, 가장 빠름
    value.store(3, std::memory_order_relaxed);
    int v3 = value.load(std::memory_order_relaxed);
}
```

## Atomic 연산

### Load와 Store

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> value{42};

    // Load
    int v1 = value.load();
    int v2 = value;  // 암시적 load

    // Store
    value.store(100);
    value = 200;  // 암시적 store

    std::cout << "Value: " << value.load() << "\n";
    return 0;
}
```

### Fetch-and-Add/Sub

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

std::atomic<int> counter{0};

void increment_100k() {
    for (int i = 0; i < 100000; ++i) {
        counter.fetch_add(1);  // 이전 값 반환
        // counter += 1 또는 ++counter와 동일
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(increment_100k);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Counter: " << counter << "\n";  // 400000
    return 0;
}
```

### Fetch-and-Or/And/Xor

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<unsigned int> flags{0};

    // 비트를 원자적으로 설정
    flags.fetch_or(0b0001);   // 비트 0 설정
    flags.fetch_or(0b0010);   // 비트 1 설정

    // 비트를 원자적으로 클리어
    flags.fetch_and(~0b0001); // 비트 0 클리어

    // 비트를 원자적으로 토글
    flags.fetch_xor(0b0010);  // 비트 1 토글

    std::cout << "Flags: " << flags << "\n";
    return 0;
}
```

### Exchange

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> value{0};

void swap_values() {
    int old_value = value.exchange(42);  // 42로 설정하고 이전 값 반환
    std::cout << "Old value: " << old_value << "\n";
}

int main() {
    std::thread t1(swap_values);
    std::thread t2(swap_values);
    t1.join();
    t2.join();
    std::cout << "Final value: " << value << "\n";
    return 0;
}
```

## Compare-and-Swap

### compare_exchange_weak

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> value{0};

void cas_increment() {
    int expected = value.load();
    while (!value.compare_exchange_weak(expected, expected + 1)) {
        // 실패 시 expected가 현재 값으로 업데이트됨
        // 재시도
    }
}

int main() {
    std::thread t1(cas_increment);
    std::thread t2(cas_increment);
    t1.join();
    t2.join();
    std::cout << "Value: " << value << "\n";  // 2
    return 0;
}
```

### compare_exchange_strong

```cpp
#include <atomic>
#include <iostream>

struct Node {
    int value;
    Node* next;
};

class LockFreeStack {
    std::atomic<Node*> head{nullptr};

public:
    void push(int value) {
        Node* new_node = new Node{value, nullptr};
        new_node->next = head.load();

        // head를 성공적으로 업데이트할 때까지 계속 시도
        while (!head.compare_exchange_strong(new_node->next, new_node)) {
            // 실패 시 new_node->next가 현재 head로 업데이트됨
        }
    }

    bool pop(int& value) {
        Node* old_head = head.load();
        while (old_head && !head.compare_exchange_strong(old_head, old_head->next)) {
            // 실패 시 old_head가 현재 head로 업데이트됨
        }

        if (old_head) {
            value = old_head->value;
            delete old_head;
            return true;
        }
        return false;
    }
};

int main() {
    LockFreeStack stack;
    stack.push(1);
    stack.push(2);
    stack.push(3);

    int value;
    while (stack.pop(value)) {
        std::cout << value << " ";
    }
    std::cout << "\n";
    return 0;
}
```

### Weak vs. Strong

```cpp
#include <atomic>

std::atomic<int> value{0};

void example_weak() {
    int expected = 0;
    // 일부 아키텍처에서 가짜 실패(spurious failure) 가능
    // 루프에서 사용
    while (!value.compare_exchange_weak(expected, 1)) {
        // 재시도
    }
}

void example_strong() {
    int expected = 0;
    // 가짜 실패 없음
    // 루프 없이 사용 가능 (실패 시 재시도하지 않는 경우)
    if (value.compare_exchange_strong(expected, 1)) {
        // 성공
    } else {
        // 실제 실패 (expected != value)
    }
}
```

## 다른 언어와의 비교

### C++ vs. C#
```cpp
// C++
std::atomic<int> counter{0};
counter.fetch_add(1);

// C# 동등 코드:
// int counter = 0;
// Interlocked.Increment(ref counter);
```

### C++ vs. Go
```cpp
// C++
std::atomic<int> counter{0};
counter.store(42);

// Go 동등 코드:
// var counter int32
// atomic.StoreInt32(&counter, 42)
```

### C++ vs. JavaScript
```cpp
// C++
std::atomic<int> counter{0};
counter.fetch_add(1);

// JavaScript (SharedArrayBuffer + Atomics):
// const buffer = new SharedArrayBuffer(4);
// const view = new Int32Array(buffer);
// Atomics.add(view, 0, 1);
```

## 모범 사례

### 1. 간단한 동기화에 Atomic 사용

```cpp
// 좋음: 간단한 카운터
std::atomic<int> counter{0};
++counter;

// 과도함: 간단한 카운터에 mutex 사용하지 않기
std::mutex mtx;
int counter = 0;
{
    std::lock_guard lock(mtx);
    ++counter;
}
```

### 2. 처음에는 순차적 일관성 선호

```cpp
// 좋음: seq_cst로 시작 (기본값)
std::atomic<int> value{0};
value.store(42);  // memory_order_seq_cst가 암시됨

// 고급: 나중에 필요하면 최적화
value.store(42, std::memory_order_release);
```

### 3. 동기화에 Acquire-Release 사용

```cpp
// 생산자-소비자 패턴
std::atomic<bool> ready{false};
int data;

void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire));
    // 여기서 data가 보임
}
```

### 4. 카운터에 Relaxed 사용 (순서가 중요하지 않을 때)

```cpp
// 좋음: 간단한 카운터에 relaxed 사용
std::atomic<long> request_count{0};

void handle_request() {
    // 단순 카운팅, 순서는 중요하지 않음
    request_count.fetch_add(1, std::memory_order_relaxed);
}
```

### 5. 타입이 Lock-Free인지 확인

```cpp
#include <atomic>
#include <iostream>

struct LargeStruct {
    long data[100];
};

int main() {
    std::atomic<int> atomic_int;
    std::atomic<LargeStruct> atomic_large;

    std::cout << "int is lock-free: "
              << atomic_int.is_lock_free() << "\n";
    std::cout << "LargeStruct is lock-free: "
              << atomic_large.is_lock_free() << "\n";

    // 또는 컴파일 타임에 (C++17)
    static_assert(std::atomic<int>::is_always_lock_free);

    return 0;
}
```

## 일반적인 실수

### 1. 모든 Atomic이 Lock-Free라고 가정하기

```cpp
// 나쁨: Lock-Free가 아닐 수 있음!
struct BigStruct {
    long data[1000];
};
std::atomic<BigStruct> big_atomic;  // 내부적으로 mutex를 사용할 수 있음!

// 좋음: 먼저 확인
if (!big_atomic.is_lock_free()) {
    std::cout << "Warning: Not lock-free!\n";
}
```

### 2. Atomic과 비원자적 접근 혼합

```cpp
// 나쁨: 데이터 경쟁!
std::atomic<int> value{0};

void thread1() {
    value.store(42);  // 원자적
}

void thread2() {
    int* ptr = reinterpret_cast<int*>(&value);
    *ptr = 100;  // 비원자적 - 정의되지 않은 동작!
}
```

### 3. 메모리 순서 잊기

```cpp
// 나쁨: relaxed가 필요한 동기화를 제공하지 않을 수 있음
std::atomic<bool> ready{false};
int data;

void producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // 너무 약함!
}

void consumer() {
    while (!ready.load(std::memory_order_relaxed));
    // data가 보이지 않을 수 있음!
}

// 좋음: acquire-release 사용
void producer_fixed() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer_fixed() {
    while (!ready.load(std::memory_order_acquire));
    // data가 보임이 보장됨
}
```

### 4. ABA 문제

```cpp
// 나쁨: ABA 문제
class Stack {
    std::atomic<Node*> head;

    void pop() {
        Node* old_head = head.load();
        // 스레드 1 여기서 일시 중지
        // 스레드 2: A를 pop, B를 pop, A를 push (같은 주소!)
        // 스레드 1 재개:
        head.compare_exchange_strong(old_head, old_head->next);
        // 성공, 하지만 B가 사라짐!
    }
};

// 해결책: 태그 포인터 또는 hazard 포인터 사용
```

### 5. False Sharing

```cpp
// 나쁨: False sharing - atomic들이 같은 캐시 라인에 있음
struct Counters {
    std::atomic<int> counter1{0};
    std::atomic<int> counter2{0};
};

// 좋음: 별도의 캐시 라인
struct Counters {
    alignas(64) std::atomic<int> counter1{0};
    alignas(64) std::atomic<int> counter2{0};
};
```

## 내부 메커니즘

### Lock-Free 보장과 구현

```cpp
// std::atomic<T>의 lock-free 여부 확인
template<typename T>
class atomic {
    // lock-free가 아닐 경우 내부 뮤텍스 사용
    alignas(T) char _M_storage[sizeof(T)];

    // 큰 타입의 경우: 글로벌 뮤텍스 테이블에서 해시
    static std::mutex& _get_lock(const void* addr) {
        constexpr size_t TABLE_SIZE = 16;
        static std::mutex table[TABLE_SIZE];
        return table[reinterpret_cast<uintptr_t>(addr) % TABLE_SIZE];
    }
};

// is_always_lock_free 컴파일 타임 체크 (C++17)
static_assert(std::atomic<int>::is_always_lock_free);
static_assert(std::atomic<long long>::is_always_lock_free);  // x86-64
// static_assert(std::atomic<__int128>::is_always_lock_free);  // 실패할 수 있음
```

**Lock-Free 크기 제한**:
| 아키텍처 | Lock-Free 보장 크기 |
|---------|---------------------|
| x86-64  | 1, 2, 4, 8 bytes (자연 정렬 시) |
| x86-64 + CMPXCHG16B | 16 bytes 가능 |
| ARM64   | 1, 2, 4, 8 bytes |
| ARM64 + LSE | 16 bytes 가능 (ldp/stp atomic) |

### 메모리 순서에서 CPU 명령어로의 매핑

**x86-64 (TSO 모델)**:
```cpp
// x86은 기본적으로 강한 순서 보장
atomic<int> x;

x.store(1, memory_order_relaxed);
// MOV [x], 1

x.store(1, memory_order_release);
// MOV [x], 1  (x86 store는 자동 release)

x.store(1, memory_order_seq_cst);
// MOV [x], 1
// MFENCE  또는  XCHG [x], 1  (full barrier 필요)

int v = x.load(memory_order_acquire);
// MOV EAX, [x]  (x86 load는 자동 acquire)

v = x.load(memory_order_seq_cst);
// MOV EAX, [x]  (load는 추가 barrier 불필요)
```

**ARM64 (Weak 모델)**:
```cpp
x.store(1, memory_order_relaxed);
// STR W0, [X1]

x.store(1, memory_order_release);
// STLR W0, [X1]  (Store-Release)

x.store(1, memory_order_seq_cst);
// STLR W0, [X1]
// DMB ISH  (또는 STLR만으로 충분할 수 있음)

int v = x.load(memory_order_acquire);
// LDAR W0, [X1]  (Load-Acquire)
```

### Fetch-and-Add 어셈블리

```cpp
// x.fetch_add(1, memory_order_relaxed)
// x86-64:
//   LOCK XADD [x], EAX
//   (LOCK prefix가 원자성 + 암시적 full barrier 제공)

// x.fetch_add(1, memory_order_acquire)
// x86-64: 동일 (LOCK은 이미 acquire 의미)
//   LOCK XADD [x], EAX

// ARM64 (LSE):
//   LDADDAL W0, W0, [X1]  (Atomic Add, Acquire-Release)
// ARM64 (non-LSE, LL/SC):
//   LDAXR W0, [X1]
//   ADD W2, W0, #1
//   STLXR W3, W2, [X1]
//   CBNZ W3, retry
```

### Compare-Exchange 구현 세부사항

```cpp
// compare_exchange_weak vs strong
atomic<int> x;
int expected = 0;

// weak: spurious failure 가능 (ARM LL/SC에서)
while (!x.compare_exchange_weak(expected, 1)) {
    expected = 0;  // 재시도
}

// strong: 값이 같으면 반드시 성공
// ARM에서는 내부적으로 루프 사용
if (x.compare_exchange_strong(expected, 1)) {
    // 성공
}
```

**x86-64 CMPXCHG**:
```asm
; compare_exchange_strong(expected, desired)
MOV EAX, expected      ; EAX = expected value
MOV ECX, desired       ; ECX = desired value
LOCK CMPXCHG [x], ECX  ; if (*x == EAX) *x = ECX; else EAX = *x;
JE success             ; ZF=1 이면 성공
; EAX에 실제 값이 저장됨 (expected 업데이트)
```

**ARM64 LL/SC (weak 구현)**:
```asm
; compare_exchange_weak
LDXR W0, [X1]          ; Load-Exclusive
CMP W0, expected
BNE fail               ; 값 다르면 실패
STXR W2, desired, [X1] ; Store-Exclusive (실패 가능!)
CBNZ W2, spurious_fail ; Exclusive 실패 = spurious failure
```

### atomic_flag: 유일하게 Lock-Free가 보장되는 것

```cpp
// atomic_flag는 항상 lock-free 보장
// 내부적으로 단순 boolean (1바이트 또는 정렬을 위해 더 클 수 있음)

struct atomic_flag {
    // 가능한 구현
    alignas(4) unsigned char _M_flag;  // 또는 int

    bool test_and_set(memory_order order) noexcept {
        // x86: LOCK BTS 또는 LOCK XCHG
        // ARM: LDAXRB + STLXRB 루프
    }

    void clear(memory_order order) noexcept {
        // x86: MOV [flag], 0  (+ MFENCE if seq_cst)
        // ARM: STLRB
    }
};

// C++20: test() 함수 추가
bool test(memory_order order) const noexcept;
```

### Atomic Reference 래퍼 (C++20)

```cpp
// atomic_ref: 기존 객체를 원자적으로 접근
int regular_int = 0;
std::atomic_ref<int> ref(regular_int);

ref.fetch_add(1);  // regular_int를 원자적으로 증가

// 제약 조건:
// 1. 객체는 atomic_ref의 수명 동안 유효해야 함
// 2. 다른 비원자적 접근과 동시에 사용하면 UB
// 3. required_alignment 정렬 필요
static_assert(alignof(int) >= std::atomic_ref<int>::required_alignment);
```

### Double-Width CAS (DWCAS)

```cpp
// 16바이트 원자적 연산 (x86-64 + CMPXCHG16B)
struct alignas(16) DoubleWord {
    uint64_t ptr;
    uint64_t counter;
};

std::atomic<DoubleWord> dw;

// CMPXCHG16B 요구사항:
// 1. 16바이트 정렬 필수
// 2. -mcx16 컴파일 옵션 필요
// 3. CPU가 CMPXCHG16B 지원해야 함

// 어셈블리:
// LOCK CMPXCHG16B [addr]
// RCX:RBX = new value
// RDX:RAX = expected value
// 성공 시 ZF=1, 실패 시 RDX:RAX = 실제 값
```

## 성능 고려사항

### 연산 비용

```cpp
// 상대적 비용 (매우 대략적):
// Relaxed atomic:  1x
// Acquire-release: 1-2x
// Seq_cst:        2-10x
// Mutex lock:     25x (비경합), 1000x+ (경합)
```

### 언제 무엇을 사용할 것인가

```cpp
// 간단한 카운터: Relaxed
std::atomic<long> stats{0};
stats.fetch_add(1, std::memory_order_relaxed);

// 데이터 의존성이 있는 플래그: Acquire-release
std::atomic<bool> ready{false};
int data;
ready.store(true, std::memory_order_release);

// 여러 atomic 변수: 순차적 일관성
std::atomic<int> x{0}, y{0};
x.store(1);  // seq_cst가 전체 순서 보장
y.store(1);

// 복잡한 데이터 구조: mutex 사용
std::mutex mtx;
ComplexStructure data;
```

## 전체 예제: Lock-Free 큐

```cpp
#include <atomic>
#include <memory>
#include <iostream>

template<typename T>
class LockFreeQueue {
    struct Node {
        std::shared_ptr<T> data;
        std::atomic<Node*> next;
        Node() : next(nullptr) {}
    };

    std::atomic<Node*> head;
    std::atomic<Node*> tail;

public:
    LockFreeQueue() {
        Node* dummy = new Node();
        head.store(dummy);
        tail.store(dummy);
    }

    ~LockFreeQueue() {
        while (Node* old_head = head.load()) {
            head.store(old_head->next);
            delete old_head;
        }
    }

    void push(T value) {
        auto data = std::make_shared<T>(std::move(value));
        Node* new_node = new Node();
        Node* old_tail = tail.load();

        while (true) {
            Node* null_ptr = nullptr;
            if (old_tail->next.compare_exchange_strong(null_ptr, new_node)) {
                old_tail->data = data;
                tail.compare_exchange_strong(old_tail, new_node);
                return;
            } else {
                tail.compare_exchange_strong(old_tail, old_tail->next.load());
            }
        }
    }

    std::shared_ptr<T> pop() {
        Node* old_head = head.load();
        while (old_head != tail.load()) {
            if (head.compare_exchange_strong(old_head, old_head->next)) {
                std::shared_ptr<T> result = old_head->next.load()->data;
                delete old_head;
                return result;
            }
        }
        return nullptr;
    }
};

int main() {
    LockFreeQueue<int> queue;

    queue.push(1);
    queue.push(2);
    queue.push(3);

    while (auto value = queue.pop()) {
        std::cout << *value << " ";
    }
    std::cout << "\n";

    return 0;
}
```

## 추가 읽기

- [C++ Reference: std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)
- [C++ 메모리 모델](https://en.cppreference.com/w/cpp/atomic/memory_order)
- "C++ Concurrency in Action" by Anthony Williams
- [Lock-Free 프로그래밍](../../05-advanced-patterns/)

## 탐색

- [C++ 개요로 돌아가기](./README.md)
- 이전: [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
- 다음: [Condition Variable](./04-condition-variable.md)
