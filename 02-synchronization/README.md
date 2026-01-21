# 동기화 기법 (Synchronization Techniques)

## 📌 개요

이 섹션에서는 멀티스레드 환경에서 **안전한 데이터 공유**를 위한 동기화 기법을 다룹니다. Race Condition을 방지하고 스레드 간 협력을 가능하게 하는 핵심 메커니즘을 학습합니다.

---

## 🎯 학습 목표

이 섹션을 완료하면 다음을 이해할 수 있습니다:

1. ✅ Mutex와 Lock을 이용한 상호 배제 (Mutual Exclusion)
2. ✅ Semaphore를 이용한 리소스 접근 제어
3. ✅ Condition Variable을 이용한 스레드 간 협력
4. ✅ Atomic Operation과 CAS를 이용한 Lock-Free 프로그래밍
5. ✅ Memory Barrier와 Memory Ordering
6. ✅ Reader-Writer Lock을 이용한 읽기/쓰기 최적화

---

## 📚 문서 목록

### [01. Mutex와 Lock](./01-mutex-lock.md)
**핵심 개념**: Mutex는 한 번에 하나의 스레드만 임계 영역(Critical Section)에 진입하도록 보장합니다.

**다루는 내용**:
- Mutex의 동작 원리
- Lock, Try-Lock, Timed-Lock
- Recursive Mutex와 일반 Mutex
- Lock Guard와 RAII 패턴
- Spinlock vs Sleeping Lock

**왜 중요한가**: 가장 기본적이고 널리 사용되는 동기화 메커니즘입니다.

---

### [02. Semaphore](./02-semaphore.md)
**핵심 개념**: Semaphore는 제한된 개수의 리소스에 대한 접근을 제어합니다.

**다루는 내용**:
- Counting Semaphore vs Binary Semaphore
- Producer-Consumer 문제
- Semaphore vs Mutex 비교
- 실전 사용 예시

**왜 중요한가**: 리소스 풀, 연결 제한 등 실전에서 자주 사용됩니다.

---

### [03. Condition Variable](./03-condition-variable.md)
**핵심 개념**: Condition Variable은 특정 조건이 만족될 때까지 스레드를 대기시키고, 조건 충족 시 깨웁니다.

**다루는 내용**:
- Wait, Notify, Broadcast
- Spurious Wakeup 문제
- Mutex와의 협력
- Producer-Consumer 패턴 구현

**왜 중요한가**: Busy-Waiting을 피하고 효율적인 스레드 협력을 구현합니다.

---

### [04. Atomic Operations](./04-atomic-operations.md)
**핵심 개념**: Atomic Operation은 중단되지 않고 완전히 실행되거나 전혀 실행되지 않는 연산입니다.

**다루는 내용**:
- Compare-And-Swap (CAS)
- Fetch-Add, Exchange
- ABA 문제
- Lock-Free 자료구조 기초

**왜 중요한가**: Lock 없이 고성능 동기화를 구현할 수 있습니다.

---

### [05. Memory Barrier](./05-memory-barrier.md)
**핵심 개념**: Memory Barrier는 메모리 연산의 순서를 보장하여 CPU 재배치를 제어합니다.

**다루는 내용**:
- Memory Reordering 문제
- Acquire-Release 시맨틱
- Sequential Consistency vs Relaxed Ordering
- Fence 명령어

**왜 중요한가**: Lock-Free 프로그래밍의 정확성을 보장합니다.

---

### [06. Reader-Writer Lock](./06-rwlock.md)
**핵심 개념**: Reader-Writer Lock은 읽기는 여러 스레드가 동시에, 쓰기는 독점적으로 수행하도록 합니다.

**다루는 내용**:
- Read Lock vs Write Lock
- Reader-Preference vs Writer-Preference
- Shared Mutex (C++17)
- 성능 최적화 전략

**왜 중요한가**: 읽기가 많은 워크로드에서 성능을 크게 향상시킵니다.

---

### [07. Futex (Fast Userspace Mutex)](./07-futex.md)
**핵심 개념**: Futex는 Linux의 저수준 동기화 프리미티브로, 유저 스페이스와 커널 스페이스를 결합한 하이브리드 메커니즘입니다.

**다루는 내용**:
- Futex의 동작 원리 (Fast Path vs Slow Path)
- Futex 기반 Mutex, Semaphore, Condition Variable 구현
- FUTEX_WAIT, FUTEX_WAKE, FUTEX_REQUEUE 연산
- Priority Inheritance (FUTEX_LOCK_PI)
- 플랫폼별 유사 메커니즘 (Windows WaitOnAddress, macOS ulock)

**왜 중요한가**: pthread mutex 등 모든 고수준 동기화 도구의 기반이며, Linux 동기화 성능의 핵심입니다.

---

## 🔍 핵심 개념 요약

### 동기화 기법 비교

| 기법 | 용도 | 성능 | 복잡도 | 사용 사례 |
|------|------|------|--------|-----------|
| **Mutex** | 상호 배제 | 중간 | 낮음 | 공유 자원 보호 |
| **Semaphore** | 리소스 카운팅 | 중간 | 중간 | 연결 풀, 리소스 제한 |
| **Condition Variable** | 조건 대기 | 높음 | 중간 | Producer-Consumer |
| **Atomic** | Lock-Free 동기화 | 매우 높음 | 높음 | 카운터, 플래그 |
| **Memory Barrier** | 순서 보장 | 높음 | 매우 높음 | Lock-Free 자료구조 |
| **RWLock** | 읽기/쓰기 분리 | 높음 (읽기 많을 때) | 중간 | 캐시, 설정 |
| **Futex** | 커널 동기화 기반 | 매우 높음 (경합 없을 때) | 매우 높음 | Mutex/Semaphore 구현 |

### 선택 가이드

```
단순 공유 자원 보호?
    └─→ Mutex + Lock Guard

제한된 리소스 관리?
    └─→ Semaphore

조건 기반 대기?
    └─→ Condition Variable + Mutex

읽기가 대부분?
    └─→ Reader-Writer Lock

최고 성능 필요? (단순 카운터, 플래그)
    └─→ Atomic Operations

Lock-Free 자료구조?
    └─→ Atomic + Memory Barrier
```

---

## 🎓 학습 경로

```
1. Mutex와 Lock (필수)
   ↓
2. Semaphore (필수)
   ↓
3. Condition Variable (필수)
   ↓
4. Atomic Operations (권장)
   ↓
5. Memory Barrier (고급)
   ↓
6. Reader-Writer Lock (권장)
   ↓
7. Futex (고급, Linux 특화)
   ↓
다음 섹션: 03-concurrency-problems/
```

**권장**: 1-3은 필수, 4-6은 성능 최적화가 필요할 때, 7은 저수준 구현을 이해하고 싶을 때 학습하세요.

---

## 💡 실전 적용 팁

### 1. 동기화 기법 선택 기준

**Mutex를 사용**:
- ✅ 간단한 공유 자원 보호
- ✅ 임계 영역이 짧음 (< 100μs)
- ✅ 대부분의 일반적인 경우

**Spinlock을 사용**:
- ✅ 임계 영역이 매우 짧음 (< 1μs)
- ✅ 컨텍스트 스위칭 비용이 더 큼
- ✅ 실시간 시스템

**Atomic을 사용**:
- ✅ 단일 변수만 업데이트
- ✅ 최고 성능 필요
- ✅ Lock-Free 알고리즘

**RWLock을 사용**:
- ✅ 읽기:쓰기 비율 > 10:1
- ✅ 임계 영역이 김 (캐시, 설정)

### 2. 일반적인 실수

❌ **Deadlock**:
```cpp
// 나쁜 예: 다른 순서로 락 획득
Thread 1: lock(A) → lock(B)
Thread 2: lock(B) → lock(A)  // Deadlock!
```

✅ **해결책**: 항상 같은 순서로 락 획득
```cpp
// 좋은 예: 일관된 순서
Thread 1: lock(A) → lock(B)
Thread 2: lock(A) → lock(B)  // OK
```

❌ **Lock 없이 공유 자원 접근**:
```cpp
int counter = 0;  // 공유 변수
void increment() {
    counter++;  // Race Condition!
}
```

✅ **해결책**:
```cpp
std::mutex mtx;
int counter = 0;
void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;
}
```

### 3. 성능 최적화

**락 경합 최소화**:
- 임계 영역을 최대한 짧게 유지
- 락 밖에서 준비 작업 수행
- 락 분할 (Lock Striping)

**락 없는 알고리즘 고려**:
- 읽기 전용 데이터: 동기화 불필요
- 단일 쓰기 스레드: Atomic으로 충분
- 복잡한 경우: Lock-Free 자료구조

---

## 📊 동기화 비용 비교

| 연산 | 시간 (대략적) |
|------|---------------|
| **읽기 (no sync)** | ~1 ns |
| **Atomic Read** | ~10 ns |
| **Atomic CAS (성공)** | ~20 ns |
| **Mutex Lock (경합 없음)** | ~25 ns |
| **Mutex Lock (경합 있음)** | ~100 ns - 1 μs |
| **Context Switch** | ~1-10 μs |

**교훈**: 불필요한 동기화는 성능을 크게 저하시킵니다.

---

## 💻 기본 예시

### Mutex 기본 사용

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int shared_data = 0;

void increment(int n) {
    for (int i = 0; i < n; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        shared_data++;
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Result: " << shared_data << std::endl;  // 200000
    return 0;
}
```

### Atomic 기본 사용

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> shared_data(0);

void increment(int n) {
    for (int i = 0; i < n; i++) {
        shared_data.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Result: " << shared_data.load() << std::endl;  // 200000
    return 0;
}
```

### Condition Variable 기본 사용

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

void producer() {
    for (int i = 0; i < 10; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(i);
        cv.notify_one();
    }
}

void consumer() {
    for (int i = 0; i < 10; i++) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []{ return !queue.empty(); });
        int value = queue.front();
        queue.pop();
        lock.unlock();
        std::cout << "Consumed: " << value << std::endl;
    }
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);

    t1.join();
    t2.join();
    return 0;
}
```

---

## 🔗 다음 단계

동기화 기법을 이해했다면:

1. [동시성 문제](../03-concurrency-problems/README.md) - Deadlock, Livelock, Starvation
2. [동시성 패턴](../04-concurrency-patterns/README.md) - Thread Pool, Actor Model
3. [Lock-Free 프로그래밍](../05-lock-free-programming/README.md) - 고급 기법

---

## 📚 참고 자료

### 온라인
- [C++ Concurrency in Action](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition)
- [The Art of Multiprocessor Programming](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1)

### 표준 문서
- [C++11 Thread Support](https://en.cppreference.com/w/cpp/thread)
- [POSIX Threads](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)

---

*동기화는 멀티스레드 프로그래밍의 핵심입니다. 각 기법의 특성을 이해하고 상황에 맞게 선택하세요!*
