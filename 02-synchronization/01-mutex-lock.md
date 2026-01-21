# Mutex와 Lock

## 📌 핵심 개념

**Mutex (Mutual Exclusion)**는 한 번에 **하나의 스레드만** 임계 영역(Critical Section)에 진입하도록 보장하는 동기화 메커니즘입니다.

**Lock**은 Mutex를 획득하는 행위이며, **Unlock**은 Mutex를 해제하는 행위입니다.

---

## 🏗️ Mutex의 동작 원리

### 기본 개념

```
Thread 1                Thread 2                Thread 3
   │                       │                       │
   │ lock(mutex) ───→ [획득]                      │
   │                       │                       │
   │ [임계 영역]           │ lock(mutex) ──→ [대기] │
   │   counter++           │      ⏳               │
   │                       │                       │ lock(mutex) ──→ [대기]
   │ unlock(mutex)         │                       │      ⏳
   │                       │                       │
   │                  [획득] ←─────────────────────┤
   │                       │                       │      ⏳
   │                  [임계 영역]                  │
   │                     counter++                 │
   │                       │                       │
   │                  unlock(mutex)                │
   │                       │                       │
   │                       │                  [획득] ←──────
   │                       │                       │
   │                       │                  [임계 영역]
   │                       │                    counter++
   │                       │                       │
   │                       │                  unlock(mutex)
```

### 상태 다이어그램

```
        Mutex State Machine

┌──────────────┐  lock()   ┌──────────────┐
│   Unlocked   │ ────────→ │   Locked     │
│  (Available) │           │ (Owned by T1)│
└──────────────┘ ←──────── └──────────────┘
                  unlock()

Other threads trying to lock():
    └─→ Blocked (waiting in queue)
```

---

## 📊 Mutex 종류 비교

| 종류 | 재진입 | 성능 | 용도 |
|------|--------|------|------|
| **일반 Mutex** | ❌ 불가 (Deadlock) | 빠름 | 일반적인 경우 |
| **Recursive Mutex** | ✅ 가능 | 느림 | 재귀 함수, 복잡한 호출 |
| **Timed Mutex** | ❌ 불가 | 중간 | 타임아웃 필요 |
| **Shared Mutex (RWLock)** | 읽기는 공유 | 중간 | 읽기가 많을 때 |

---

## 💻 C++ 기본 사용법

### 1. std::mutex 기본

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int shared_counter = 0;

void increment(int n) {
    for (int i = 0; i < n; i++) {
        mtx.lock();           // 락 획득
        shared_counter++;     // 임계 영역
        mtx.unlock();         // 락 해제
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter << std::endl;  // 200000
    return 0;
}
```

**문제점**: 예외 발생 시 `unlock()`이 호출되지 않아 Deadlock 가능!

### 2. RAII 패턴: std::lock_guard

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int shared_counter = 0;

void increment(int n) {
    for (int i = 0; i < n; i++) {
        std::lock_guard<std::mutex> lock(mtx);  // 생성자에서 lock()
        shared_counter++;
        // 소멸자에서 자동으로 unlock()
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter << std::endl;  // 200000
    return 0;
}
```

**장점**: 예외 안전성 (Exception Safety) 보장

### 3. std::unique_lock (유연한 락)

```cpp
#include <mutex>
#include <thread>

std::mutex mtx;
int shared_data = 0;

void process() {
    std::unique_lock<std::mutex> lock(mtx);  // 락 획득

    // 임계 영역 1
    shared_data++;

    lock.unlock();  // 명시적 해제 가능

    // 락 없는 작업 (시간 소모적인 작업)
    expensive_computation();

    lock.lock();  // 다시 획득 가능

    // 임계 영역 2
    shared_data++;

    // 소멸자에서 자동 해제
}
```

**차이점**: `lock_guard`는 생성 시에만 락, `unique_lock`은 중간에 락/언락 가능

### 4. std::scoped_lock (C++17, 여러 Mutex)

```cpp
#include <mutex>

std::mutex mtx1, mtx2;
int data1 = 0, data2 = 0;

void transfer() {
    // 두 Mutex를 동시에 획득 (Deadlock 방지)
    std::scoped_lock lock(mtx1, mtx2);

    data1--;
    data2++;
}
```

**장점**: 여러 Mutex를 Deadlock 없이 획득

---

## 🔄 Recursive Mutex

### 문제 상황

```cpp
std::mutex mtx;

void inner_function() {
    std::lock_guard<std::mutex> lock(mtx);
    // ...
}

void outer_function() {
    std::lock_guard<std::mutex> lock(mtx);  // 락 획득
    inner_function();  // ❌ Deadlock! (같은 스레드가 다시 락 시도)
}
```

### 해결책: Recursive Mutex

```cpp
#include <mutex>

std::recursive_mutex rec_mtx;

void inner_function() {
    std::lock_guard<std::recursive_mutex> lock(rec_mtx);
    // ...
}

void outer_function() {
    std::lock_guard<std::recursive_mutex> lock(rec_mtx);  // 락 획득 (count = 1)
    inner_function();  // ✅ OK! (count = 2)
    // outer 끝 (count = 1)
    // inner 끝 (count = 0, 해제)
}
```

**주의**: Recursive Mutex는 느리므로 꼭 필요한 경우에만 사용

---

## ⏱️ Timed Mutex

### try_lock() - 즉시 반환

```cpp
#include <mutex>

std::mutex mtx;

void try_lock_example() {
    if (mtx.try_lock()) {
        // 락 획득 성공
        std::cout << "Lock acquired" << std::endl;
        // ... 임계 영역 ...
        mtx.unlock();
    } else {
        // 락 획득 실패 (다른 스레드가 사용 중)
        std::cout << "Lock busy, doing something else" << std::endl;
    }
}
```

### try_lock_for() - 타임아웃

```cpp
#include <mutex>
#include <chrono>

std::timed_mutex tmtx;

void timed_lock_example() {
    if (tmtx.try_lock_for(std::chrono::milliseconds(100))) {
        // 100ms 내에 락 획득 성공
        std::cout << "Lock acquired within 100ms" << std::endl;
        // ... 임계 영역 ...
        tmtx.unlock();
    } else {
        // 타임아웃
        std::cout << "Timeout!" << std::endl;
    }
}
```

**사용 사례**:
- 비즈니스 로직에서 일정 시간 이상 대기하면 안 되는 경우
- UI 스레드에서 무한 대기 방지

---

## 🔒 Spinlock vs Sleeping Mutex

### Spinlock (바쁜 대기)

```cpp
#include <atomic>

class Spinlock {
private:
    std::atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // CPU에서 계속 대기 (Busy-Wait)
        }
    }

    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

**특징**:
- ✅ 컨텍스트 스위칭 없음 (매우 빠름)
- ❌ CPU 시간 낭비
- **사용**: 임계 영역이 매우 짧을 때 (< 1μs)

### Sleeping Mutex (대기 큐)

```cpp
// std::mutex는 내부적으로 Sleeping 메커니즘 사용
std::mutex mtx;

void example() {
    mtx.lock();  // 획득 실패 시 스레드를 대기 상태로 전환
    // ... 임계 영역 ...
    mtx.unlock();
}
```

**특징**:
- ✅ CPU 시간 절약
- ❌ 컨텍스트 스위칭 비용 (~1-10μs)
- **사용**: 임계 영역이 길 때 (> 10μs)

### 비교표

| 특성 | Spinlock | Sleeping Mutex |
|------|----------|----------------|
| **대기 방식** | CPU에서 계속 확인 | 대기 큐에서 sleep |
| **컨텍스트 스위칭** | 없음 | 있음 |
| **CPU 사용** | 높음 | 낮음 |
| **지연 시간** | 매우 짧음 (~10ns) | 짧음 (~100ns) |
| **임계 영역** | 매우 짧을 때 (< 1μs) | 긴 경우 (> 10μs) |
| **사용 환경** | 실시간 시스템, 커널 | 일반 애플리케이션 |

---

## ⚠️ 주의사항 및 함정

### 1. Deadlock

```cpp
// ❌ 나쁜 예: 다른 순서로 락 획득
std::mutex mtx_a, mtx_b;

void thread1() {
    std::lock_guard<std::mutex> lock_a(mtx_a);
    std::lock_guard<std::mutex> lock_b(mtx_b);
    // ...
}

void thread2() {
    std::lock_guard<std::mutex> lock_b(mtx_b);  // 순서가 반대!
    std::lock_guard<std::mutex> lock_a(mtx_a);
    // Deadlock 발생 가능!
}
```

**해결책 1**: 일관된 순서
```cpp
// ✅ 좋은 예: 항상 같은 순서
void thread1() {
    std::lock_guard<std::mutex> lock_a(mtx_a);
    std::lock_guard<std::mutex> lock_b(mtx_b);
}

void thread2() {
    std::lock_guard<std::mutex> lock_a(mtx_a);  // 같은 순서
    std::lock_guard<std::mutex> lock_b(mtx_b);
}
```

**해결책 2**: std::scoped_lock (C++17)
```cpp
void thread1() {
    std::scoped_lock lock(mtx_a, mtx_b);  // 자동으로 Deadlock 방지
}

void thread2() {
    std::scoped_lock lock(mtx_b, mtx_a);  // 순서 상관없음
}
```

### 2. 예외 안전성

```cpp
// ❌ 나쁜 예: 예외 발생 시 unlock 안됨
std::mutex mtx;

void dangerous() {
    mtx.lock();

    risky_operation();  // 예외 발생 가능!

    mtx.unlock();  // 예외 발생 시 실행 안됨 → Deadlock!
}
```

```cpp
// ✅ 좋은 예: RAII 패턴 사용
void safe() {
    std::lock_guard<std::mutex> lock(mtx);

    risky_operation();  // 예외 발생해도 소멸자에서 unlock
}
```

### 3. 락 범위 최소화

```cpp
// ❌ 나쁜 예: 불필요하게 긴 임계 영역
std::mutex mtx;
std::vector<int> shared_data;

void process() {
    std::lock_guard<std::mutex> lock(mtx);

    // 준비 작업 (락 필요 없음)
    int result = expensive_computation();

    // 공유 자원 접근 (락 필요)
    shared_data.push_back(result);
}
```

```cpp
// ✅ 좋은 예: 최소한의 임계 영역
void process() {
    // 락 없이 준비 작업
    int result = expensive_computation();

    // 필요한 부분만 락
    {
        std::lock_guard<std::mutex> lock(mtx);
        shared_data.push_back(result);
    }
}
```

### 4. Lock Convoys (락 행렬)

```cpp
// 문제: 모든 스레드가 같은 락을 대기
std::mutex mtx;

void worker() {
    while (true) {
        std::lock_guard<std::mutex> lock(mtx);
        process_item();
        // 락 해제 → 다음 스레드가 즉시 획득 → 반복
    }
}
```

**해결책**: 락 분할 (Lock Striping)
```cpp
// 여러 개의 락을 사용
std::array<std::mutex, 16> mutexes;

void worker(int item_id) {
    int idx = item_id % 16;
    std::lock_guard<std::mutex> lock(mutexes[idx]);
    process_item(item_id);
}
```

---

## 📊 성능 고려사항

### 락 경합 (Lock Contention)

```
낮은 경합 (좋음):
Time ──────────────────────────────────>
T1:  [Lock][Work][Unlock]
T2:                      [Lock][Work][Unlock]
T3:                                         [Lock][Work][Unlock]

높은 경합 (나쁨):
Time ──────────────────────────────────>
T1:  [Lock][Work][Unlock]
T2:  [Wait...........][Lock][Work][Unlock]
T3:  [Wait..............................][Lock][Work][Unlock]
                ↑
            성능 저하!
```

### 최적화 전략

1. **임계 영역 최소화**
   - 락 밖에서 준비 작업 수행
   - 락 안에서는 최소한의 작업만

2. **락 분할 (Lock Striping)**
   - 큰 자료구조를 여러 부분으로 나누고 각각 다른 락 사용
   - 예: Java의 `ConcurrentHashMap`

3. **읽기 최적화**
   - 읽기가 많으면 Reader-Writer Lock 사용
   - 읽기 전용 데이터는 동기화 불필요

4. **Lock-Free 대안 고려**
   - 단순 카운터: `std::atomic`
   - 플래그: `std::atomic_flag`

---

## 🔗 다음 단계

- [Semaphore](./02-semaphore.md) - 제한된 리소스 관리
- [Condition Variable](./03-condition-variable.md) - 조건 기반 대기
- [Atomic Operations](./04-atomic-operations.md) - Lock-Free 프로그래밍
- [Reader-Writer Lock](./06-rwlock.md) - 읽기/쓰기 최적화

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams, Chapter 3
- [cppreference: std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)
- [POSIX Threads: pthread_mutex](https://man7.org/linux/man-pages/man3/pthread_mutex_lock.3.html)

---

*Mutex는 가장 기본적인 동기화 도구입니다. RAII 패턴과 함께 사용하여 안전한 멀티스레드 프로그램을 작성하세요!*
