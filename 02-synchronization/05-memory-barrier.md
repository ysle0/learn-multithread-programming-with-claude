# Memory Barrier (메모리 장벽)

## 📌 핵심 개념

**Memory Barrier (Fence)**는 컴파일러와 CPU의 **메모리 재배치(Reordering)**를 제어하여 메모리 연산의 순서를 보장하는 메커니즘입니다.

**핵심 문제**:
- **컴파일러 최적화**: 코드 순서를 재배치
- **CPU 재배치**: Out-of-Order 실행, Store Buffer
- **캐시 일관성**: 다른 코어에서 변경 사항이 즉시 보이지 않음

**해결책**: Memory Barrier를 통한 순서 제어

---

## 🏗️ Memory Reordering의 문제

### 문제 1: 컴파일러 재배치

```cpp
int data = 0;
bool ready = false;

void producer() {
    data = 42;      // A
    ready = true;   // B
}

void consumer() {
    while (!ready); // C
    assert(data == 42); // D
}
```

**예상**: A → B → C → D

**실제 (최적화 후)**:
```cpp
// 컴파일러가 재배치 가능!
void producer() {
    ready = true;   // B (먼저!)
    data = 42;      // A
}
// consumer에서 assert 실패 가능!
```

### 문제 2: CPU 재배치

```cpp
// Thread 1
x = 1;  // Store x
r1 = y; // Load y

// Thread 2
y = 1;  // Store y
r2 = x; // Load x

// CPU 재배치로 인해 가능한 결과: r1 = 0, r2 = 0
// (모든 Store가 Load 후에 실행)
```

### 실제 동작 방식

```
컴파일러 최적화:
Source Code → [재배치] → Assembly Code

CPU 실행:
Assembly Code → [Out-of-Order] → Store Buffer → Cache → Memory

          ┌─────────────────┐
          │   CPU Core 1    │
          ├─────────────────┤
          │ Registers       │
          │ Store Buffer    │ ← Store가 여기서 대기!
          ├─────────────────┤
          │   L1 Cache      │
          └────────┬────────┘
                   │
          ┌────────┴────────┐
          │   L2 Cache      │
          └────────┬────────┘
                   │
          ┌────────┴────────┐
          │  Main Memory    │
          └─────────────────┘

문제: Store가 즉시 메모리에 반영되지 않음!
```

---

## 💻 Memory Ordering 모델

### C++ Memory Ordering

```cpp
namespace std {
    enum memory_order {
        memory_order_relaxed,   // 순서 보장 없음
        memory_order_consume,   // Deprecated
        memory_order_acquire,   // Load 이후 재배치 방지
        memory_order_release,   // Store 이전 재배치 방지
        memory_order_acq_rel,   // Acquire + Release
        memory_order_seq_cst    // Sequential Consistency (기본값)
    };
}
```

### 1. Relaxed (순서 보장 없음)

```cpp
std::atomic<int> x(0), y(0);
int r1, r2;

// Thread 1
x.store(1, std::memory_order_relaxed);
r1 = y.load(std::memory_order_relaxed);

// Thread 2
y.store(1, std::memory_order_relaxed);
r2 = x.load(std::memory_order_relaxed);

// 가능: r1 = 0, r2 = 0 (재배치 허용)
```

**특징**:
- ✅ 원자성만 보장
- ❌ 순서 보장 없음
- **사용**: 단순 카운터, 통계

### 2. Acquire (읽기 장벽)

```cpp
std::atomic<bool> ready(false);
int data = 0;

// Thread 1
data = 42;                                    // A
ready.store(true, std::memory_order_release); // B

// Thread 2
while (!ready.load(std::memory_order_acquire)); // C
assert(data == 42);                             // D

// 보장:
// - C 이후의 모든 읽기/쓰기는 C 앞으로 이동 불가
// - A → B → C → D
```

**Acquire의 의미**:
```
    acquire
       ↓
  ┌─────────┐
  │ Barrier │  ← 이 아래로 이동 불가
  └─────────┘
       ↓
  [읽기/쓰기]
```

### 3. Release (쓰기 장벽)

```cpp
std::atomic<bool> ready(false);
int data = 0;

// Thread 1
data = 42;                                    // A
ready.store(true, std::memory_order_release); // B

// 보장:
// - B 이전의 모든 읽기/쓰기는 B 뒤로 이동 불가
// - A가 B보다 먼저 실행
```

**Release의 의미**:
```
  [읽기/쓰기]
       ↓
  ┌─────────┐
  │ Barrier │  ← 이 위로 이동 불가
  └─────────┘
       ↓
    release
```

### 4. Acquire-Release (양방향 장벽)

```cpp
std::atomic<int> x(0), y(0);

// Thread 1
x.store(1, std::memory_order_release);
int r1 = y.load(std::memory_order_acquire);

// Thread 2
y.store(1, std::memory_order_release);
int r2 = x.load(std::memory_order_acquire);

// 보장: r1 = 0 또는 r2 = 0 (최소 하나는 0)
// 불가능: r1 = 0, r2 = 0 (동시에 0은 불가)
```

### 5. Sequential Consistency (전체 순서)

```cpp
std::atomic<int> x(0), y(0);
int r1, r2;

// Thread 1
x.store(1);  // memory_order_seq_cst (기본값)
r1 = y.load();

// Thread 2
y.store(1);
r2 = x.load();

// 보장: 모든 스레드가 같은 순서로 관찰
// 불가능: r1 = 0, r2 = 0
```

---

## 🎯 실전 사용 예시

### 1. Acquire-Release: Producer-Consumer

```cpp
#include <atomic>
#include <thread>
#include <cassert>

std::atomic<int*> ptr(nullptr);

void producer() {
    int* data = new int(42);

    // data를 완전히 초기화한 후 ptr 설정
    ptr.store(data, std::memory_order_release);
}

void consumer() {
    int* data;

    // ptr이 설정될 때까지 대기
    while (!(data = ptr.load(std::memory_order_acquire)));

    assert(*data == 42);  // 항상 성공
    delete data;
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);

    t1.join();
    t2.join();
    return 0;
}
```

### 2. Relaxed: 카운터

```cpp
#include <atomic>
#include <thread>
#include <vector>

std::atomic<long long> counter(0);

void increment() {
    for (int i = 0; i < 100000; i++) {
        counter.fetch_add(1, std::memory_order_relaxed);
        // 순서가 중요하지 않음 (단순 증가)
    }
}

int main() {
    std::vector<std::thread> threads;

    for (int i = 0; i < 10; i++) {
        threads.emplace_back(increment);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Counter: " << counter.load() << std::endl;
    return 0;
}
```

### 3. Seq_Cst: 복잡한 동기화

```cpp
#include <atomic>
#include <thread>

std::atomic<bool> x(false), y(false);
std::atomic<int> z(0);

void write_x() {
    x.store(true);  // seq_cst
}

void write_y() {
    y.store(true);  // seq_cst
}

void read_x_then_y() {
    while (!x.load());  // seq_cst
    if (y.load()) {     // seq_cst
        z.fetch_add(1);
    }
}

void read_y_then_x() {
    while (!y.load());  // seq_cst
    if (x.load()) {     // seq_cst
        z.fetch_add(1);
    }
}

int main() {
    std::thread t1(write_x);
    std::thread t2(write_y);
    std::thread t3(read_x_then_y);
    std::thread t4(read_y_then_x);

    t1.join(); t2.join(); t3.join(); t4.join();

    // z는 항상 1 또는 2 (0은 불가능)
    assert(z.load() >= 1);
    return 0;
}
```

### 4. std::atomic_thread_fence (명시적 Fence)

```cpp
#include <atomic>

std::atomic<bool> ready(false);
int data = 0;

void producer() {
    data = 42;

    // Fence: data 쓰기가 ready 쓰기보다 먼저 보이도록 보장
    std::atomic_thread_fence(std::memory_order_release);

    ready.store(true, std::memory_order_relaxed);
}

void consumer() {
    while (!ready.load(std::memory_order_relaxed));

    // Fence: ready 읽기가 data 읽기보다 먼저 보이도록 보장
    std::atomic_thread_fence(std::memory_order_acquire);

    assert(data == 42);
}
```

---

## 🔍 하드웨어 레벨 이해

### CPU 메모리 모델

#### x86-64 (강한 모델)

```
x86-64 보장:
- Store-Load 재배치만 가능
- Load-Load, Load-Store, Store-Store는 재배치 안됨

따라서 Acquire-Release가 대부분 no-op (fence 명령 불필요)
```

#### ARM/PowerPC (약한 모델)

```
ARM 특징:
- 모든 재배치 가능
- 명시적 Fence 명령 필요 (DMB, DSB, ISB)

따라서 Acquire-Release에서 실제 fence 명령 실행
```

### Fence 명령어

```cpp
// x86-64
void x86_fence() {
    asm volatile("mfence" ::: "memory");  // Full fence
    asm volatile("lfence" ::: "memory");  // Load fence
    asm volatile("sfence" ::: "memory");  // Store fence
}

// ARM
void arm_fence() {
    asm volatile("dmb ish" ::: "memory"); // Data Memory Barrier
    asm volatile("dsb ish" ::: "memory"); // Data Sync Barrier
    asm volatile("isb" ::: "memory");     // Instruction Sync Barrier
}
```

---

## 📊 성능 비교

### Memory Ordering 성능

```cpp
// 벤치마크 (100만 번 atomic 증가)

// Relaxed: ~5ms
for (int i = 0; i < 1000000; i++) {
    counter.fetch_add(1, std::memory_order_relaxed);
}

// Acquire-Release: ~10ms
for (int i = 0; i < 1000000; i++) {
    counter.fetch_add(1, std::memory_order_acq_rel);
}

// Sequential Consistency: ~20ms
for (int i = 0; i < 1000000; i++) {
    counter.fetch_add(1, std::memory_order_seq_cst);
}
```

### 아키텍처별 차이

| Ordering | x86-64 | ARM | 비고 |
|----------|--------|-----|------|
| **relaxed** | ~5ms | ~5ms | 동일 |
| **acquire/release** | ~6ms | ~15ms | ARM에서 fence 필요 |
| **seq_cst** | ~20ms | ~30ms | 모두 느림 |

**결론**: 약한 메모리 모델(ARM)에서 Ordering 비용이 더 큼

---

## ⚠️ 주의사항 및 함정

### 1. Data Race

```cpp
// ❌ 나쁜 예: Atomic 없이 공유 변수 접근
std::atomic<bool> ready(false);
int data = 0;  // 일반 변수!

// Thread 1
data = 42;
ready.store(true, std::memory_order_relaxed);  // Relaxed!

// Thread 2
if (ready.load(std::memory_order_relaxed)) {
    std::cout << data << std::endl;  // Data Race! (UB)
}
```

```cpp
// ✅ 좋은 예: Release-Acquire
// Thread 1
data = 42;
ready.store(true, std::memory_order_release);

// Thread 2
if (ready.load(std::memory_order_acquire)) {
    std::cout << data << std::endl;  // OK
}
```

### 2. 잘못된 Ordering 조합

```cpp
// ❌ 나쁜 예: Release-Relaxed 조합
// Thread 1
data = 42;
ready.store(true, std::memory_order_release);

// Thread 2
if (ready.load(std::memory_order_relaxed)) {  // Relaxed!
    std::cout << data << std::endl;  // Data Race 가능!
}
```

**규칙**: Release와 Acquire는 쌍으로 사용

### 3. Fence 위치 오류

```cpp
// ❌ 나쁜 예: Fence 후 atomic 연산
data = 42;
std::atomic_thread_fence(std::memory_order_release);
ready.store(true, std::memory_order_seq_cst);  // 더 강한 ordering!
// Fence가 무의미함
```

```cpp
// ✅ 좋은 예: Fence 후 relaxed atomic
data = 42;
std::atomic_thread_fence(std::memory_order_release);
ready.store(true, std::memory_order_relaxed);
```

---

## 🔍 고급 주제

### 1. Happens-Before 관계

```cpp
// Happens-Before: A → B (A가 B보다 먼저 발생)

// 규칙 1: 같은 스레드 내 순서
a = 1;  // A
b = 2;  // B
// A → B (프로그램 순서)

// 규칙 2: Release-Acquire
// Thread 1
data = 42;                                   // A
ready.store(true, std::memory_order_release); // B

// Thread 2
if (ready.load(std::memory_order_acquire)) {  // C
    assert(data == 42);                       // D
}
// A → B, B synchronizes-with C, C → D
// 따라서 A → D (Happens-Before)
```

### 2. Load-Buffering 문제

```cpp
std::atomic<int> x(0), y(0);
int r1, r2;

// Thread 1
r1 = x.load(std::memory_order_seq_cst);
y.store(1, std::memory_order_seq_cst);

// Thread 2
r2 = y.load(std::memory_order_seq_cst);
x.store(1, std::memory_order_seq_cst);

// Sequential Consistency: r1 = 0, r2 = 0 불가능
// Acquire-Release: r1 = 0, r2 = 0 가능!
```

### 3. Consume Ordering (Deprecated)

```cpp
// C++17부터 Deprecated (구현 어려움)
// Acquire보다 약한 ordering (데이터 의존성만 보장)

std::atomic<int*> ptr(nullptr);

// Thread 1
int* data = new int(42);
ptr.store(data, std::memory_order_release);

// Thread 2
int* p = ptr.load(std::memory_order_consume);  // Deprecated!
if (p) {
    std::cout << *p << std::endl;  // 의존성 있음 (OK)
}

// 현재는 acquire로 대체 권장
```

---

## 🔗 다음 단계

- [Atomic Operations](./04-atomic-operations.md) - Atomic 복습
- [Lock-Free 프로그래밍](../05-lock-free-programming/README.md) - 실전 적용
- [Reader-Writer Lock](./06-rwlock.md) - 다른 동기화 기법

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams, Chapter 5
- [cppreference: Memory Order](https://en.cppreference.com/w/cpp/atomic/memory_order)
- [Preshing: Memory Ordering](https://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
- [Linux Kernel Memory Barriers](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

---

*Memory Barrier는 Lock-Free 프로그래밍의 핵심입니다. 의심스러울 때는 Sequential Consistency를 사용하세요!*
