# Semaphore (세마포어)

## 📌 핵심 개념

**Semaphore**는 **제한된 개수의 리소스**에 대한 접근을 제어하는 동기화 메커니즘입니다. 내부적으로 카운터를 유지하며, 카운터가 0보다 크면 접근을 허용하고 0이면 대기시킵니다.

**핵심 연산**:
- **wait() (P, acquire)**: 카운터 감소, 0이면 대기
- **signal() (V, release)**: 카운터 증가, 대기 중인 스레드 깨움

---

## 🏗️ Semaphore의 동작 원리

### 기본 개념

```
Semaphore(3)  ← 최대 3개 리소스

Thread 1: wait()  → count: 3 → 2 [허용]
Thread 2: wait()  → count: 2 → 1 [허용]
Thread 3: wait()  → count: 1 → 0 [허용]
Thread 4: wait()  → count: 0 → 0 [대기] ⏳
Thread 5: wait()  → count: 0 → 0 [대기] ⏳

Thread 1: signal() → count: 0 → 1 [Thread 4 깨움]
Thread 4: [실행]
Thread 2: signal() → count: 0 → 1 [Thread 5 깨움]
Thread 5: [실행]
```

### 상태 다이어그램

```
        Semaphore(count = 3)

┌─────────────────────────────────────┐
│  Available Count: 3                 │
│  ┌───┐ ┌───┐ ┌───┐                 │
│  │ ✓ │ │ ✓ │ │ ✓ │                 │
│  └───┘ └───┘ └───┘                 │
└─────────────────────────────────────┘

   wait()              wait()
Thread 1 ────→ ✓       Thread 4 ──→ [Blocked]
Thread 2 ────→ ✓       Thread 5 ──→ [Blocked]
Thread 3 ────→ ✓

   signal()
Thread 1 ────→ [Thread 4 깨움]
```

---

## 📊 Semaphore 종류

### 1. Binary Semaphore (이진 세마포어)

카운터가 0 또는 1만 가능 (Mutex와 유사)

```cpp
// 초기값 1
Semaphore sem(1);

void critical_section() {
    sem.wait();    // count: 1 → 0
    // 임계 영역
    sem.signal();  // count: 0 → 1
}
```

### 2. Counting Semaphore (카운팅 세마포어)

카운터가 여러 값 가능 (리소스 풀)

```cpp
// 초기값 5 (5개 연결 허용)
Semaphore sem(5);

void use_connection() {
    sem.wait();    // count: 5 → 4 → 3 → ...
    // 연결 사용
    sem.signal();  // count: 4 → 5
}
```

### Semaphore vs Mutex

| 특성 | Semaphore | Mutex |
|------|-----------|-------|
| **목적** | 리소스 카운팅 | 상호 배제 |
| **초기값** | 임의의 양수 | 1 (Binary) |
| **소유권** | ❌ 없음 (누구나 signal 가능) | ✅ 있음 (lock한 스레드만 unlock) |
| **사용 사례** | 리소스 풀, Producer-Consumer | 임계 영역 보호 |
| **재진입** | 불가 | Recursive Mutex는 가능 |

---

## 🔧 내부 구현 메커니즘

### P(wait) 연산 원자적 구현

```c
// 개념적 구현 (원자적으로 실행)
void sem_wait(sem_t *sem) {
    while (true) {
        int count = atomic_load(&sem->count);
        if (count > 0) {
            if (atomic_compare_exchange_weak(&sem->count, &count, count - 1)) {
                return;  // 성공
            }
            // CAS 실패, 재시도
        } else {
            // count == 0, 커널에서 대기
            futex_wait(&sem->count, 0);
        }
    }
}
```

### V(signal) 연산 원자적 구현

```c
// 개념적 구현
void sem_post(sem_t *sem) {
    int old_count = atomic_fetch_add(&sem->count, 1);

    // 대기자가 있을 수 있으면 깨움
    if (old_count == 0) {
        futex_wake(&sem->count, 1);  // 한 스레드 깨우기
    }
}
```

### POSIX sem_t 내부 구조 (Linux glibc)

```c
// glibc의 sem_t 구조 (단순화)
typedef struct {
    unsigned int value;     // 현재 카운트 값
    int private;            // 프로세스 간 공유 여부
    // 하위 비트: 실제 값
    // 상위 비트: 대기자 수 (최적화용)
} sem_t;
```

### 명명된(Named) vs 무명(Unnamed) 세마포어

```
Named Semaphore:
┌─────────────────────────────────────────┐
│ Process A              Process B        │
│ sem_open("/my_sem")    sem_open("/my_sem")
│         ↓                    ↓          │
│     ┌──────────────────────────┐        │
│     │  /dev/shm/sem.my_sem    │ ← 공유 │
│     │  (메모리 매핑 파일)       │        │
│     └──────────────────────────┘        │
└─────────────────────────────────────────┘

Unnamed Semaphore:
┌─────────────────────────────────────────┐
│ 같은 프로세스 내 스레드들 공유           │
│ sem_init(&sem, 0, initial_value)       │
│         pshared=0: 스레드 간만 공유     │
│         pshared=1: mmap 영역에서 프로세스 간│
└─────────────────────────────────────────┘
```

### 플랫폼별 구현 차이

| 플랫폼 | 내부 구현 | 특징 |
|--------|----------|------|
| **Linux** | Futex 기반 | Fast path는 user-space atomic |
| **macOS** | dispatch_semaphore (GCD) | Mach 커널 세마포어 래핑 |
| **Windows** | 커널 오브젝트 (HANDLE) | 항상 커널 모드 진입 |
| **FreeBSD** | umtx 기반 | Linux futex 유사 |

### Linux POSIX Semaphore 상세 동작

```c
// Linux sem_wait 내부 (단순화)
int sem_wait(sem_t *sem) {
    unsigned int *futex_addr = &sem->value;

    while (1) {
        unsigned int val = atomic_load(futex_addr);

        // Fast path: count > 0
        if (likely(val > 0)) {
            if (atomic_cmpxchg(futex_addr, val, val - 1) == val)
                return 0;  // 성공
            continue;  // 재시도
        }

        // Slow path: count == 0, futex 대기
        futex(futex_addr, FUTEX_WAIT_PRIVATE, 0, NULL);
    }
}

// Linux sem_post 내부 (단순화)
int sem_post(sem_t *sem) {
    unsigned int *futex_addr = &sem->value;

    // count 증가
    unsigned int old = atomic_fetch_add(futex_addr, 1);

    // 대기자가 있을 가능성이 있으면 깨움
    // (성능 최적화: 상위 비트로 대기자 존재 추적)
    if (likely(old == 0)) {
        futex(futex_addr, FUTEX_WAKE_PRIVATE, 1);
    }

    return 0;
}
```

### 성능 특성

| 연산 | Uncontended | Contended |
|------|-------------|-----------|
| **sem_wait** | ~50ns (atomic만) | ~1-10μs (커널 대기) |
| **sem_post** | ~50ns | ~50ns (wake 포함) |
| **sem_trywait** | ~20ns | ~20ns |

---

## 💻 C++ 구현 (C++20)

### C++20 std::counting_semaphore

```cpp
#include <semaphore>
#include <thread>
#include <iostream>
#include <vector>

// 최대 3개 동시 접근 허용
std::counting_semaphore<3> sem(3);

void worker(int id) {
    sem.acquire();  // wait()
    std::cout << "Worker " << id << " using resource" << std::endl;
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Worker " << id << " releasing resource" << std::endl;
    sem.release();  // signal()
}

int main() {
    std::vector<std::thread> threads;

    // 10개 스레드 생성 (최대 3개만 동시 실행)
    for (int i = 0; i < 10; i++) {
        threads.emplace_back(worker, i);
    }

    for (auto& t : threads) {
        t.join();
    }

    return 0;
}
```

### C++20 std::binary_semaphore

```cpp
#include <semaphore>

// Binary Semaphore (0 또는 1)
std::binary_semaphore sem(1);

void critical_section() {
    sem.acquire();
    // 임계 영역
    sem.release();
}
```

### C++11/14/17: POSIX Semaphore (Unix/Linux)

```cpp
#include <semaphore.h>
#include <pthread.h>

sem_t sem;

// 초기화
sem_init(&sem, 0, 3);  // 0: 스레드 간 공유, 3: 초기값

// 사용
sem_wait(&sem);    // wait()
// ... 작업 ...
sem_post(&sem);    // signal()

// 정리
sem_destroy(&sem);
```

### 직접 구현 (C++11, Mutex + Condition Variable)

```cpp
#include <mutex>
#include <condition_variable>

class Semaphore {
private:
    std::mutex mtx;
    std::condition_variable cv;
    int count;

public:
    explicit Semaphore(int initial_count) : count(initial_count) {}

    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return count > 0; });
        count--;
    }

    void signal() {
        std::lock_guard<std::mutex> lock(mtx);
        count++;
        cv.notify_one();
    }

    bool try_wait() {
        std::lock_guard<std::mutex> lock(mtx);
        if (count > 0) {
            count--;
            return true;
        }
        return false;
    }
};
```

---

## 🎯 실전 사용 예시

### 1. 연결 풀 제한 (Connection Pool)

```cpp
#include <semaphore>
#include <thread>
#include <iostream>
#include <vector>

// 최대 10개 데이터베이스 연결
std::counting_semaphore<10> db_connections(10);

void database_query(int query_id) {
    db_connections.acquire();  // 연결 획득

    std::cout << "Query " << query_id << " executing..." << std::endl;
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    std::cout << "Query " << query_id << " done" << std::endl;

    db_connections.release();  // 연결 해제
}

int main() {
    std::vector<std::thread> queries;

    // 100개 쿼리 (최대 10개만 동시 실행)
    for (int i = 0; i < 100; i++) {
        queries.emplace_back(database_query, i);
    }

    for (auto& t : queries) {
        t.join();
    }

    return 0;
}
```

### 2. Producer-Consumer 문제

```cpp
#include <semaphore>
#include <thread>
#include <queue>
#include <iostream>

const int BUFFER_SIZE = 5;

std::queue<int> buffer;
std::binary_semaphore mutex(1);          // 상호 배제
std::counting_semaphore<BUFFER_SIZE> empty(BUFFER_SIZE);  // 빈 슬롯
std::counting_semaphore<BUFFER_SIZE> full(0);             // 찬 슬롯

void producer(int id) {
    for (int i = 0; i < 10; i++) {
        int item = id * 100 + i;

        empty.acquire();   // 빈 슬롯 대기
        mutex.acquire();   // 버퍼 잠금

        buffer.push(item);
        std::cout << "Producer " << id << " produced " << item << std::endl;

        mutex.release();   // 버퍼 해제
        full.release();    // 찬 슬롯 증가
    }
}

void consumer(int id) {
    for (int i = 0; i < 10; i++) {
        full.acquire();    // 찬 슬롯 대기
        mutex.acquire();   // 버퍼 잠금

        int item = buffer.front();
        buffer.pop();
        std::cout << "Consumer " << id << " consumed " << item << std::endl;

        mutex.release();   // 버퍼 해제
        empty.release();   // 빈 슬롯 증가
    }
}

int main() {
    std::thread p1(producer, 1);
    std::thread c1(consumer, 1);

    p1.join();
    c1.join();

    return 0;
}
```

### 3. 스레드 풀 작업 제한

```cpp
#include <semaphore>
#include <thread>
#include <vector>
#include <functional>

class ThreadPool {
private:
    std::counting_semaphore<100> max_tasks;  // 최대 100개 작업
    std::vector<std::thread> workers;

public:
    ThreadPool(int num_threads) : max_tasks(100) {
        for (int i = 0; i < num_threads; i++) {
            workers.emplace_back([this] { worker_thread(); });
        }
    }

    void submit_task(std::function<void()> task) {
        max_tasks.acquire();  // 작업 슬롯 획득
        // 작업 큐에 추가
        max_tasks.release();  // 작업 완료 시 해제
    }

private:
    void worker_thread() {
        // 작업 처리
    }
};
```

### 4. Rate Limiting (속도 제한)

```cpp
#include <semaphore>
#include <thread>
#include <chrono>

class RateLimiter {
private:
    std::counting_semaphore<100> tokens;  // 초당 100개 토큰

public:
    RateLimiter() : tokens(100) {
        // 토큰 재충전 스레드
        std::thread([this] {
            while (true) {
                std::this_thread::sleep_for(std::chrono::seconds(1));
                // 초당 100개 토큰 재충전
                for (int i = 0; i < 100; i++) {
                    tokens.release();
                }
            }
        }).detach();
    }

    void acquire_token() {
        tokens.acquire();  // 토큰 획득 (없으면 대기)
    }
};

void api_call(RateLimiter& limiter, int id) {
    limiter.acquire_token();
    std::cout << "API call " << id << std::endl;
}
```

---

## 🔍 고급 패턴

### 1. Readers-Writers Problem (Semaphore로 구현)

```cpp
#include <semaphore>
#include <mutex>

std::binary_semaphore write_sem(1);  // Writer 상호 배제
std::mutex reader_count_mtx;
int reader_count = 0;

void reader() {
    // Reader 진입
    {
        std::lock_guard<std::mutex> lock(reader_count_mtx);
        reader_count++;
        if (reader_count == 1) {
            write_sem.acquire();  // 첫 Reader가 Writer 차단
        }
    }

    // 읽기 수행
    read_data();

    // Reader 퇴장
    {
        std::lock_guard<std::mutex> lock(reader_count_mtx);
        reader_count--;
        if (reader_count == 0) {
            write_sem.release();  // 마지막 Reader가 Writer 허용
        }
    }
}

void writer() {
    write_sem.acquire();  // Writer 독점
    write_data();
    write_sem.release();
}
```

### 2. Barrier (동기화 장벽)

```cpp
#include <semaphore>
#include <mutex>

class Barrier {
private:
    int count;
    int threshold;
    std::mutex mtx;
    std::binary_semaphore sem1{0};
    std::binary_semaphore sem2{1};

public:
    explicit Barrier(int num_threads) : count(0), threshold(num_threads) {}

    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        count++;

        if (count == threshold) {
            // 마지막 스레드: 모두 깨움
            sem2.acquire();
            sem1.release();
        }

        lock.unlock();

        // 모든 스레드가 도착할 때까지 대기
        sem1.acquire();
        sem1.release();

        lock.lock();
        count--;

        if (count == 0) {
            sem1.acquire();
            sem2.release();
        }
    }
};

void worker(Barrier& barrier, int id) {
    std::cout << "Thread " << id << " phase 1" << std::endl;

    barrier.wait();  // 모든 스레드 대기

    std::cout << "Thread " << id << " phase 2" << std::endl;
}
```

---

## ⚠️ 주의사항 및 함정

### 1. Deadlock

```cpp
// ❌ 나쁜 예: Semaphore로 Deadlock
std::binary_semaphore sem_a(1);
std::binary_semaphore sem_b(1);

void thread1() {
    sem_a.acquire();
    sem_b.acquire();  // Thread 2가 sem_b 보유 시 Deadlock!
}

void thread2() {
    sem_b.acquire();
    sem_a.acquire();  // Thread 1이 sem_a 보유 시 Deadlock!
}
```

**해결책**: 일관된 순서로 획득
```cpp
// ✅ 좋은 예
void thread1() {
    sem_a.acquire();
    sem_b.acquire();
}

void thread2() {
    sem_a.acquire();  // 같은 순서
    sem_b.acquire();
}
```

### 2. 소유권 문제

```cpp
// ⚠️ Semaphore는 소유권이 없음
std::binary_semaphore sem(1);

void thread1() {
    sem.acquire();
    // ... 작업 ...
    // unlock 없이 종료!
}

void thread2() {
    sem.release();  // 다른 스레드가 release 가능 (위험!)
}
```

**교훈**: 상호 배제는 Mutex, 리소스 카운팅은 Semaphore

### 3. 초기값 오류

```cpp
// ❌ 나쁜 예: 잘못된 초기값
std::counting_semaphore<5> sem(0);  // 초기값 0

void worker() {
    sem.acquire();  // 모든 스레드가 영원히 대기!
    // ...
}
```

```cpp
// ✅ 좋은 예: 올바른 초기값
std::counting_semaphore<5> sem(5);  // 리소스 5개
```

### 4. Signal without Wait

```cpp
// ⚠️ wait 없이 signal 호출
std::counting_semaphore<10> sem(5);

void buggy() {
    sem.release();  // count: 5 → 6 → 7 → ...
    sem.release();
    sem.release();
    // count가 최대값을 초과할 수 있음!
}
```

---

## 📊 성능 고려사항

### Semaphore vs Mutex 성능

| 연산 | Mutex | Semaphore |
|------|-------|-----------|
| **Lock/Acquire** | ~25 ns | ~50 ns |
| **Unlock/Release** | ~25 ns | ~50 ns |
| **경합 있을 때** | ~100 ns - 1 μs | ~100 ns - 1 μs |

**결론**: Semaphore가 약간 느리므로 단순 상호 배제는 Mutex 사용

### 최적화 팁

1. **Binary Semaphore 대신 Mutex**
   ```cpp
   // ❌ 느림
   std::binary_semaphore sem(1);

   // ✅ 빠름
   std::mutex mtx;
   ```

2. **적절한 초기값 설정**
   - 리소스 개수에 맞게 설정
   - 너무 크면 메모리 낭비

3. **Semaphore 재사용**
   - 매번 생성/삭제하지 말고 재사용

---

## 🔗 다음 단계

- [Condition Variable](./03-condition-variable.md) - 더 효율적인 대기 메커니즘
- [Mutex와 Lock](./01-mutex-lock.md) - 상호 배제 복습
- [동시성 패턴](../04-concurrency-patterns/README.md) - Producer-Consumer 패턴

---

## 📖 참고 자료

- "Operating Systems: Three Easy Pieces" - Chapter 31 (Semaphores)
- [C++20 Semaphore](https://en.cppreference.com/w/cpp/thread/counting_semaphore)
- "The Little Book of Semaphores" - Allen B. Downey

---

*Semaphore는 리소스 풀 관리에 유용하지만, 단순한 상호 배제는 Mutex를 사용하세요!*
