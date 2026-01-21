# Atomic Operations (원자적 연산)

## 📌 핵심 개념

**Atomic Operation**은 **중단되지 않고** 완전히 실행되거나 전혀 실행되지 않는 연산입니다. 다른 스레드에서는 중간 상태를 볼 수 없습니다.

**핵심 특징**:
- **불가분성(Atomicity)**: 중간 상태 없이 한 번에 실행
- **가시성(Visibility)**: 변경 사항이 다른 스레드에 즉시 보임
- **순서(Ordering)**: Memory Ordering을 통한 실행 순서 제어

**장점**: Lock 없이 동기화 가능 (Lock-Free)

---

## 🏗️ Atomic의 필요성

### 문제: Race Condition

```cpp
// ❌ 비원자적 연산
int counter = 0;

void increment() {
    counter++;  // 실제로는 3단계:
                // 1. 메모리에서 counter 읽기
                // 2. 레지스터에서 +1
                // 3. 메모리에 쓰기
}

// Thread 1과 Thread 2가 동시 실행:
// T1: read(0) → add(1) → write(1)
// T2: read(0) → add(1) → write(1)
// 결과: 2가 아니라 1!
```

### 해결책 1: Mutex (느림)

```cpp
std::mutex mtx;
int counter = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);  // ~25ns
    counter++;
}
```

### 해결책 2: Atomic (빠름)

```cpp
std::atomic<int> counter(0);

void increment() {
    counter++;  // 원자적 연산, ~10ns
}
```

---

## 💻 C++ std::atomic 기본 사용법

### 1. 기본 타입

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> counter(0);
std::atomic<bool> flag(false);
std::atomic<double> value(0.0);  // C++20부터 float/double 지원

void worker() {
    counter.fetch_add(1);  // counter++
    flag.store(true);
    double v = value.load();
}

int main() {
    std::thread t1(worker);
    std::thread t2(worker);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter.load() << std::endl;  // 2
    return 0;
}
```

### 2. 주요 연산

```cpp
std::atomic<int> x(0);

// 읽기/쓰기
int val = x.load();           // 읽기
x.store(42);                  // 쓰기
int old = x.exchange(100);    // 교환 (old = 42, x = 100)

// 산술 연산
x.fetch_add(5);               // x += 5, 이전 값 반환
x.fetch_sub(3);               // x -= 3
x++;                          // fetch_add(1)과 동일
x--;                          // fetch_sub(1)과 동일

// 비트 연산
x.fetch_and(0xFF);            // x &= 0xFF
x.fetch_or(0x10);             // x |= 0x10
x.fetch_xor(0x01);            // x ^= 0x01
```

### 3. Compare-And-Swap (CAS)

```cpp
std::atomic<int> x(100);

// compare_exchange_strong: 비교 후 교환
int expected = 100;
bool success = x.compare_exchange_strong(expected, 200);

if (success) {
    // x가 100이었으면 200으로 변경됨
    std::cout << "Success! x = 200" << std::endl;
} else {
    // x가 100이 아니었으면 expected에 현재 값 저장
    std::cout << "Failed! x = " << expected << std::endl;
}
```

**동작**:
```cpp
// compare_exchange_strong의 의미:
if (x == expected) {
    x = desired;
    return true;
} else {
    expected = x;
    return false;
}
// 위 과정이 원자적으로 수행됨!
```

### 4. compare_exchange_weak vs strong

```cpp
std::atomic<int> x(0);
int expected = 0;

// weak: Spurious failure 가능 (루프에서 사용)
while (!x.compare_exchange_weak(expected, 1)) {
    expected = 0;  // 재시도
}

// strong: Spurious failure 없음 (단일 시도)
if (x.compare_exchange_strong(expected, 1)) {
    // 성공
}
```

| 함수 | Spurious Failure | 성능 | 사용 |
|------|------------------|------|------|
| **weak** | 가능 | 빠름 | 루프 안 |
| **strong** | 없음 | 느림 | 단일 시도 |

---

## 🎯 실전 사용 예시

### 1. Lock-Free 카운터

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

std::atomic<long long> counter(0);

void increment(int n) {
    for (int i = 0; i < n; i++) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::vector<std::thread> threads;

    for (int i = 0; i < 10; i++) {
        threads.emplace_back(increment, 100000);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Counter: " << counter.load() << std::endl;  // 1000000
    return 0;
}
```

### 2. Lock-Free 플래그 (Spinlock)

```cpp
#include <atomic>

class Spinlock {
private:
    std::atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Busy-wait
        }
    }

    void unlock() {
        flag.clear(std::memory_order_release);
    }
};

Spinlock spinlock;

void critical_section() {
    spinlock.lock();
    // ... 임계 영역 ...
    spinlock.unlock();
}
```

### 3. Lock-Free Stack (단순 버전)

```cpp
#include <atomic>

template<typename T>
class LockFreeStack {
private:
    struct Node {
        T data;
        Node* next;
        Node(const T& d) : data(d), next(nullptr) {}
    };

    std::atomic<Node*> head{nullptr};

public:
    void push(const T& data) {
        Node* new_node = new Node(data);
        new_node->next = head.load();

        // CAS로 head 업데이트
        while (!head.compare_exchange_weak(new_node->next, new_node)) {
            // 실패 시 재시도
        }
    }

    bool pop(T& result) {
        Node* old_head = head.load();

        while (old_head && !head.compare_exchange_weak(old_head, old_head->next)) {
            // 실패 시 재시도
        }

        if (old_head) {
            result = old_head->data;
            delete old_head;  // ⚠️ ABA 문제 있음!
            return true;
        }
        return false;
    }
};
```

### 4. Double-Checked Locking (싱글톤)

```cpp
#include <atomic>
#include <mutex>

class Singleton {
private:
    static std::atomic<Singleton*> instance;
    static std::mutex mtx;

    Singleton() {}

public:
    static Singleton* get_instance() {
        Singleton* tmp = instance.load(std::memory_order_acquire);

        if (tmp == nullptr) {
            std::lock_guard<std::mutex> lock(mtx);
            tmp = instance.load(std::memory_order_relaxed);

            if (tmp == nullptr) {
                tmp = new Singleton();
                instance.store(tmp, std::memory_order_release);
            }
        }

        return tmp;
    }
};

std::atomic<Singleton*> Singleton::instance{nullptr};
std::mutex Singleton::mtx;
```

**C++11 이후 더 나은 방법**:
```cpp
class Singleton {
public:
    static Singleton& get_instance() {
        static Singleton instance;  // 스레드 안전 보장
        return instance;
    }
};
```

---

## 🔍 Memory Ordering (메모리 순서)

### Memory Ordering 종류

```cpp
std::atomic<int> x(0);

// 1. memory_order_relaxed: 순서 보장 없음 (가장 빠름)
x.store(1, std::memory_order_relaxed);

// 2. memory_order_acquire: 이후 읽기/쓰기가 앞으로 이동 불가
x.load(std::memory_order_acquire);

// 3. memory_order_release: 이전 읽기/쓰기가 뒤로 이동 불가
x.store(1, std::memory_order_release);

// 4. memory_order_acq_rel: acquire + release
x.exchange(1, std::memory_order_acq_rel);

// 5. memory_order_seq_cst: 전체 순서 보장 (기본값, 가장 느림)
x.store(1, std::memory_order_seq_cst);
```

### Relaxed (순서 보장 없음)

```cpp
std::atomic<int> x(0), y(0);
int r1, r2;

// Thread 1
x.store(1, std::memory_order_relaxed);
r1 = y.load(std::memory_order_relaxed);

// Thread 2
y.store(1, std::memory_order_relaxed);
r2 = x.load(std::memory_order_relaxed);

// 가능한 결과: r1 = 0, r2 = 0 (재배치 때문!)
```

**사용**: 단순 카운터 (순서가 중요하지 않을 때)

### Acquire-Release (동기화)

```cpp
std::atomic<bool> ready(false);
int data = 0;

// Thread 1 (Producer)
data = 42;                                    // A
ready.store(true, std::memory_order_release); // B

// Thread 2 (Consumer)
while (!ready.load(std::memory_order_acquire)); // C
assert(data == 42);                             // D

// 보장: A → B → C → D (data = 42 보장)
```

**사용**: Producer-Consumer, 플래그 기반 동기화

### Sequential Consistency (전체 순서)

```cpp
std::atomic<int> x(0), y(0);
int r1, r2;

// Thread 1
x.store(1);  // memory_order_seq_cst (기본값)
r1 = y.load();

// Thread 2
y.store(1);
r2 = x.load();

// 불가능: r1 = 0, r2 = 0 (전체 순서 보장)
```

**사용**: 가장 안전 (기본값), 복잡한 동기화

### 성능 비교

| Ordering | 성능 | 보장 | 사용 사례 |
|----------|------|------|-----------|
| **relaxed** | 가장 빠름 | 원자성만 | 카운터, 통계 |
| **acquire/release** | 중간 | Happens-Before | Producer-Consumer |
| **seq_cst** | 가장 느림 | 전체 순서 | 복잡한 동기화 |

---

## ⚠️ 주의사항 및 함정

### 1. ABA 문제

```cpp
// Lock-Free Stack에서 ABA 문제
std::atomic<Node*> head;

// Thread 1: pop()
Node* old_head = head.load();  // A
// [Thread 2가 A, B를 pop하고 A를 다시 push]
head.compare_exchange_strong(old_head, old_head->next);  // A' (성공!)
// 문제: A와 A'는 다른 노드인데 포인터 값이 같음!
```

**해결책**: Tagged Pointer 또는 Hazard Pointer

```cpp
struct TaggedPointer {
    Node* ptr;
    uintptr_t tag;  // 버전 번호
};

std::atomic<TaggedPointer> head;

void push(Node* node) {
    TaggedPointer old_head = head.load();
    TaggedPointer new_head;

    do {
        node->next = old_head.ptr;
        new_head.ptr = node;
        new_head.tag = old_head.tag + 1;  // 태그 증가
    } while (!head.compare_exchange_weak(old_head, new_head));
}
```

### 2. 잘못된 Memory Ordering

```cpp
// ❌ 나쁜 예: Data Race
std::atomic<bool> ready(false);
int data = 0;  // 일반 변수

// Thread 1
data = 42;
ready.store(true, std::memory_order_relaxed);  // 순서 보장 안됨!

// Thread 2
if (ready.load(std::memory_order_relaxed)) {
    std::cout << data << std::endl;  // 42가 아닐 수 있음!
}
```

```cpp
// ✅ 좋은 예: Acquire-Release
// Thread 1
data = 42;
ready.store(true, std::memory_order_release);

// Thread 2
if (ready.load(std::memory_order_acquire)) {
    std::cout << data << std::endl;  // 항상 42
}
```

### 3. Atomic이 아닌 복합 연산

```cpp
std::atomic<int> x(0);

// ❌ 원자적이지 않음!
if (x == 0) {
    x = 1;
}
// Thread 1: if (x == 0)  → true
// Thread 2: if (x == 0)  → true
// Thread 1: x = 1
// Thread 2: x = 1  (중복!)
```

```cpp
// ✅ CAS 사용
int expected = 0;
while (!x.compare_exchange_weak(expected, 1)) {
    if (expected != 0) break;  // 이미 설정됨
    expected = 0;
}
```

### 4. False Sharing

```cpp
// ❌ 나쁜 예: Cache Line 공유
struct Data {
    std::atomic<int> counter1;  // Cache Line 1
    std::atomic<int> counter2;  // Cache Line 1 (같은 라인!)
};

// Thread 1이 counter1 수정 → Cache Line 무효화
// Thread 2가 counter2 수정 → Cache Line 무효화
// 성능 저하!
```

```cpp
// ✅ 좋은 예: Cache Line 분리
struct alignas(64) Data {  // 64 바이트 정렬 (일반적인 Cache Line 크기)
    std::atomic<int> counter1;
    char padding[60];  // 패딩
};

Data data1;  // Cache Line 1
Data data2;  // Cache Line 2 (분리됨)
```

---

## 📊 성능 비교

### Atomic vs Mutex

```cpp
// 벤치마크 (100만 번 증가)

// Mutex: ~50ms
std::mutex mtx;
int counter = 0;
for (int i = 0; i < 1000000; i++) {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;
}

// Atomic (seq_cst): ~20ms
std::atomic<int> counter(0);
for (int i = 0; i < 1000000; i++) {
    counter.fetch_add(1);
}

// Atomic (relaxed): ~5ms
for (int i = 0; i < 1000000; i++) {
    counter.fetch_add(1, std::memory_order_relaxed);
}
```

**결론**: Atomic이 Mutex보다 2-10배 빠름

---

## 🔗 다음 단계

- [Memory Barrier](./05-memory-barrier.md) - Memory Ordering 심화
- [Lock-Free 프로그래밍](../05-lock-free-programming/README.md) - 고급 자료구조
- [Mutex와 Lock](./01-mutex-lock.md) - 기본 동기화 복습

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams, Chapter 5
- [cppreference: std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)
- [Preshing on Programming: Memory Ordering](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)

---

*Atomic Operation은 Lock-Free 프로그래밍의 핵심입니다. Memory Ordering을 이해하고 올바르게 사용하세요!*
