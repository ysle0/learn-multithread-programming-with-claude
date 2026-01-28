# Condition Variable (조건 변수)

## 📌 핵심 개념

**Condition Variable**은 특정 조건이 만족될 때까지 스레드를 **효율적으로 대기**시키고, 조건이 충족되면 깨우는 동기화 메커니즘입니다.

**핵심 연산**:
- **wait()**: 조건이 만족될 때까지 대기
- **notify_one()**: 대기 중인 스레드 하나를 깨움
- **notify_all()**: 대기 중인 모든 스레드를 깨움

**중요**: Condition Variable은 항상 **Mutex와 함께** 사용해야 합니다.

---

## 🏗️ Condition Variable의 동작 원리

### Busy-Waiting vs Condition Variable

#### ❌ Busy-Waiting (비효율적)

```cpp
std::mutex mtx;
bool ready = false;

void bad_wait() {
    while (true) {
        mtx.lock();
        if (ready) {
            mtx.unlock();
            break;
        }
        mtx.unlock();
        // CPU 시간 낭비!
    }
}
```

**문제점**: CPU를 계속 사용하며 조건을 확인 (매우 비효율적)

#### ✅ Condition Variable (효율적)

```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void good_wait() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });  // ready가 true일 때까지 대기
    // ready == true, 작업 수행
}

void notify() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // 대기 중인 스레드 깨움
}
```

**장점**: 대기 중에는 CPU를 사용하지 않음

### 상태 다이어그램

```
Thread 1 (Consumer)              Thread 2 (Producer)
      │                                 │
      │ lock(mtx)                       │
      │ check: ready == false           │
      │ wait(cv) ──────────┐            │
      │ [unlock mtx]       │            │
      │ [Sleep] ⏳         │            │
      │                    │            │ lock(mtx)
      │                    │            │ ready = true
      │                    │            │ unlock(mtx)
      │                    │            │ notify_one(cv)
      │ [Wakeup]           │            │
      │ [lock mtx] ←───────┘            │
      │ check: ready == true            │
      │ [proceed]                       │
      │ unlock(mtx)                     │
```

---

## 💻 C++ 기본 사용법

### 1. std::condition_variable (기본)

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void worker() {
    std::unique_lock<std::mutex> lock(mtx);
    std::cout << "Worker waiting..." << std::endl;

    cv.wait(lock, []{ return ready; });  // ready == true까지 대기

    std::cout << "Worker proceeding!" << std::endl;
}

void signal_ready() {
    std::this_thread::sleep_for(std::chrono::seconds(1));

    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }

    cv.notify_one();  // worker 깨움
}

int main() {
    std::thread t1(worker);
    std::thread t2(signal_ready);

    t1.join();
    t2.join();
    return 0;
}
```

### 2. wait() 내부 동작 이해

```cpp
// cv.wait(lock, predicate)는 다음과 같이 동작:

while (!predicate()) {
    cv.wait(lock);  // 1. unlock(mtx)
                    // 2. sleep (대기 큐)
                    // 3. notify 받으면 wakeup
                    // 4. lock(mtx)
}
```

**중요**: `wait()`는 **Spurious Wakeup**을 방지하기 위해 Predicate(조건)를 반복 확인합니다.

---

## 🔧 내부 구현 메커니즘

### Linux pthread_cond 구조

```c
// glibc pthread_cond_t 내부 (단순화)
typedef struct {
    unsigned int __wseq;        // Waiter sequence number
    unsigned int __g1_start;    // Group 1 시작
    unsigned int __g_refs[2];   // Group 참조 카운트
    unsigned int __g_size[2];   // Group 크기
    unsigned int __g_signals;   // 시그널 수
    unsigned int __wrefs;       // Writer 참조
    // ...
} pthread_cond_t;
```

### wait() 원자적 동작 (Mutex unlock + Sleep)

```c
// pthread_cond_wait 내부 동작 (단순화)
int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex) {
    // 1. 현재 sequence 저장
    unsigned int seq = atomic_load(&cond->__wseq);

    // 2. Mutex 해제 (원자적 시작점)
    pthread_mutex_unlock(mutex);

    // 3. Futex 대기 (seq가 변하면 깨어남)
    //    이 시점에서 unlock과 sleep이 원자적으로 보장됨
    futex_wait(&cond->__wseq, seq);

    // 4. Mutex 재획득
    pthread_mutex_lock(mutex);

    return 0;
}
```

**원자성 보장 원리**:
```
Thread 1 (waiter):              Thread 2 (signaler):
   seq = load(__wseq)
   unlock(mutex)                 lock(mutex)
   futex_wait(__wseq, seq)       modify_shared_data()
      ↓                          unlock(mutex)
      │                          __wseq++
      │                          futex_wake(__wseq)
      ←─────────────────────────────┘
   lock(mutex)
```

### signal vs broadcast 내부 구현

```c
// pthread_cond_signal
int pthread_cond_signal(pthread_cond_t *cond) {
    // Sequence 번호 증가
    atomic_fetch_add(&cond->__wseq, 1);

    // Futex wake: 한 스레드만 깨움
    futex_wake(&cond->__wseq, 1);
    return 0;
}

// pthread_cond_broadcast
int pthread_cond_broadcast(pthread_cond_t *cond) {
    atomic_fetch_add(&cond->__wseq, 1);

    // 모든 대기자 깨움
    futex_wake(&cond->__wseq, INT_MAX);
    return 0;
}
```

### FUTEX_REQUEUE 최적화 (Thundering Herd 방지)

```c
// 최적화된 broadcast (glibc 실제 구현)
int optimized_broadcast(pthread_cond_t *cond, pthread_mutex_t *mutex) {
    atomic_fetch_add(&cond->__wseq, 1);

    // 한 스레드만 깨우고, 나머지는 mutex의 futex로 이동
    // → Thundering Herd 방지
    futex_requeue(&cond->__wseq,    // 원래 대기 위치
                  1,                 // 깨울 스레드 수
                  INT_MAX,           // requeue할 스레드 수
                  &mutex->__lock);   // 새 대기 위치
    return 0;
}
```

**Thundering Herd 문제**:
```
일반 broadcast:
cond에서 10개 스레드 깨움 → 10개 모두 mutex 경쟁 → 1개 획득, 9개 다시 sleep

REQUEUE 최적화:
cond에서 1개만 깨움 → 나머지 9개는 mutex futex로 직접 이동
→ 불필요한 context switching 감소
```

### Spurious Wakeup 원인

**1. 시그널/인터럽트**:
```c
// 커널이 EINTR로 futex_wait 중단 가능
while (futex_wait(...) == -EINTR) {
    // 재시도 필요
}
```

**2. 구현 최적화**:
```c
// 일부 구현에서 성능상 이유로 추가 스레드 깨움 가능
// → 항상 조건 재확인 필요
```

**3. FUTEX_WAKE 타이밍**:
```c
// wake 호출 시점과 실제 깨어나는 시점 차이
// → 조건이 다시 바뀔 수 있음
```

### Mesa vs Hoare 의미론

```
Mesa 의미론 (대부분의 구현):
┌─────────────────────────────────────────┐
│ Signaler                  Waiter        │
│    │                         │          │
│    │ signal() ───────────→ [wake]       │
│    │ [계속 실행]             │           │
│    │     ↓                   │          │
│    │ unlock()                │          │
│    │                         ↓          │
│    │                    [mutex lock]    │
│    │                    [조건 재확인!]   │
└─────────────────────────────────────────┘

Hoare 의미론 (이론적):
┌─────────────────────────────────────────┐
│ Signaler                  Waiter        │
│    │                         │          │
│    │ signal() ──────────→ [즉시 실행]   │
│    │ [suspend]               │          │
│    │                         ↓          │
│    │                    [조건 보장됨]    │
└─────────────────────────────────────────┘
```

**실제 구현이 Mesa를 사용하는 이유**:
- 구현이 훨씬 간단
- Hoare는 signal 호출자를 suspend해야 함
- 대부분의 OS 스케줄러와 호환

### 성능 특성

| 연산 | Uncontended | Contended |
|------|-------------|-----------|
| **wait (sleep)** | - | ~1-2μs |
| **wait (wake)** | - | ~1-2μs |
| **notify_one** | ~50ns | ~100ns |
| **notify_all** | ~50ns | ~N×100ns |

### 3. notify_one() vs notify_all()

```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void worker(int id) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });
    std::cout << "Worker " << id << " running" << std::endl;
}

void signal_one() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // 하나만 깨움
}

void signal_all() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_all();  // 모두 깨움
}
```

| 함수 | 동작 | 사용 사례 |
|------|------|-----------|
| **notify_one()** | 대기 중인 스레드 하나만 깨움 | 리소스가 하나만 준비됨 |
| **notify_all()** | 대기 중인 모든 스레드를 깨움 | 모든 스레드가 진행 가능 |

---

## 🎯 실전 사용 예시

### 1. Producer-Consumer 패턴

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;
const int MAX_SIZE = 10;

void producer(int id) {
    for (int i = 0; i < 20; i++) {
        std::unique_lock<std::mutex> lock(mtx);

        // 큐가 가득 차면 대기
        cv.wait(lock, []{ return queue.size() < MAX_SIZE; });

        int item = id * 100 + i;
        queue.push(item);
        std::cout << "Producer " << id << " produced " << item << std::endl;

        lock.unlock();
        cv.notify_all();  // Consumer 깨움
    }
}

void consumer(int id) {
    for (int i = 0; i < 20; i++) {
        std::unique_lock<std::mutex> lock(mtx);

        // 큐가 비어있으면 대기
        cv.wait(lock, []{ return !queue.empty(); });

        int item = queue.front();
        queue.pop();
        std::cout << "Consumer " << id << " consumed " << item << std::endl;

        lock.unlock();
        cv.notify_all();  // Producer 깨움
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

### 2. 스레드 풀 작업 큐

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <functional>
#include <vector>

class ThreadPool {
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;

    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;

public:
    ThreadPool(int num_threads) {
        for (int i = 0; i < num_threads; i++) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;

                    {
                        std::unique_lock<std::mutex> lock(mtx);

                        // 작업이 있거나 stop될 때까지 대기
                        cv.wait(lock, [this]{ return stop || !tasks.empty(); });

                        if (stop && tasks.empty()) {
                            return;
                        }

                        task = std::move(tasks.front());
                        tasks.pop();
                    }

                    task();  // 작업 실행
                }
            });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            stop = true;
        }
        cv.notify_all();

        for (auto& worker : workers) {
            worker.join();
        }
    }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            tasks.push(std::move(task));
        }
        cv.notify_one();  // Worker 하나 깨움
    }
};

int main() {
    ThreadPool pool(4);

    for (int i = 0; i < 10; i++) {
        pool.enqueue([i] {
            std::cout << "Task " << i << " executing" << std::endl;
        });
    }

    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 0;
}
```

### 3. 동기화 포인트 (Barrier)

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <vector>

class Barrier {
private:
    std::mutex mtx;
    std::condition_variable cv;
    int count = 0;
    int threshold;
    int generation = 0;  // Spurious wakeup 방지

public:
    explicit Barrier(int num_threads) : threshold(num_threads) {}

    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        int gen = generation;

        if (++count == threshold) {
            // 마지막 스레드
            generation++;
            count = 0;
            cv.notify_all();
        } else {
            // 다른 스레드들 대기
            cv.wait(lock, [this, gen]{ return gen != generation; });
        }
    }
};

void worker(Barrier& barrier, int id) {
    std::cout << "Thread " << id << " phase 1" << std::endl;

    barrier.wait();  // 모든 스레드 대기

    std::cout << "Thread " << id << " phase 2" << std::endl;
}

int main() {
    Barrier barrier(5);
    std::vector<std::thread> threads;

    for (int i = 0; i < 5; i++) {
        threads.emplace_back(worker, std::ref(barrier), i);
    }

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### 4. 타임아웃 대기 (wait_for)

```cpp
#include <mutex>
#include <condition_variable>
#include <chrono>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void worker() {
    std::unique_lock<std::mutex> lock(mtx);

    // 최대 2초 대기
    if (cv.wait_for(lock, std::chrono::seconds(2), []{ return ready; })) {
        std::cout << "Condition met!" << std::endl;
    } else {
        std::cout << "Timeout!" << std::endl;
    }
}
```

---

## ⚠️ 주의사항 및 함정

### 1. Spurious Wakeup (가짜 깨어남)

```cpp
// ❌ 나쁜 예: Predicate 없이 wait
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock);  // Spurious wakeup 가능!

if (ready) {
    // ready가 false일 수도 있음!
    process();
}
```

```cpp
// ✅ 좋은 예: Predicate 사용
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, []{ return ready; });  // 조건 재확인

// ready == true 보장
process();
```

**원인**: OS 레벨에서 신호 처리, 인터럽트 등으로 인해 notify 없이도 깨어날 수 있음

### 2. Lost Wakeup (깨움 놓침)

```cpp
// ❌ 나쁜 예: notify 먼저 호출
void bad_sequence() {
    cv.notify_one();  // 이 시점에 대기 중인 스레드 없음!

    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });  // 영원히 대기!
}
```

```cpp
// ✅ 좋은 예: 조건 변경 후 notify
void good_sequence() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;  // 조건 변경
    }
    cv.notify_one();  // 그 다음 notify
}
```

### 3. 잘못된 Mutex 타입

```cpp
// ❌ 나쁜 예: lock_guard 사용
std::lock_guard<std::mutex> lock(mtx);  // unlock 불가!
cv.wait(lock);  // 컴파일 에러!
```

```cpp
// ✅ 좋은 예: unique_lock 사용
std::unique_lock<std::mutex> lock(mtx);  // unlock 가능
cv.wait(lock);  // OK
```

**이유**: `wait()`는 내부적으로 unlock/lock을 수행하므로 `unique_lock` 필요

### 4. Deadlock (notify 없이 대기)

```cpp
// ❌ 나쁜 예: notify 호출 잊음
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void worker() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, []{ return ready; });  // 영원히 대기!
}

void main() {
    std::thread t(worker);
    // notify_one() 호출 안함!
    t.join();  // Deadlock!
}
```

```cpp
// ✅ 좋은 예: notify 호출
void main() {
    std::thread t(worker);

    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // 반드시 호출!

    t.join();
}
```

### 5. notify 전에 unlock

```cpp
// ⚠️ 비효율적: lock 상태에서 notify
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
    cv.notify_one();  // 깨어난 스레드가 즉시 lock 대기
}  // unlock

// ✅ 효율적: unlock 후 notify
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}  // unlock
cv.notify_one();  // 깨어난 스레드가 바로 진행 가능
```

---

## 📊 성능 고려사항

### Condition Variable vs Busy-Waiting

| 방법 | CPU 사용 | 응답 시간 | 적합한 경우 |
|------|----------|-----------|-------------|
| **Busy-Waiting** | 매우 높음 | 매우 짧음 (~10ns) | 대기 시간 < 1μs |
| **Condition Variable** | 낮음 | 짧음 (~1μs) | 대기 시간 > 10μs |

### 최적화 팁

1. **notify 전 unlock**
   - 깨어난 스레드가 즉시 진행 가능
   - Lock contention 감소

2. **notify_one vs notify_all**
   - 하나만 처리 가능: `notify_one()` (효율적)
   - 모두 처리 가능: `notify_all()` (필요 시만)

3. **Predicate 최적화**
   - 간단한 조건 사용 (bool, counter 등)
   - 복잡한 연산은 피함

---

## 🔍 고급 주제

### 1. condition_variable_any

```cpp
// std::condition_variable: std::unique_lock<std::mutex>만 가능
std::condition_variable cv;

// std::condition_variable_any: 모든 Lock 타입 가능
std::condition_variable_any cv_any;

std::shared_mutex sh_mtx;

void example() {
    std::shared_lock<std::shared_mutex> lock(sh_mtx);
    cv_any.wait(lock);  // OK
}
```

**차이점**: `condition_variable_any`는 유연하지만 약간 느림

### 2. wait_until (절대 시간 대기)

```cpp
#include <chrono>

auto deadline = std::chrono::system_clock::now() + std::chrono::seconds(5);

std::unique_lock<std::mutex> lock(mtx);
if (cv.wait_until(lock, deadline, []{ return ready; })) {
    std::cout << "Condition met before deadline" << std::endl;
} else {
    std::cout << "Deadline reached" << std::endl;
}
```

---

## 🔗 다음 단계

- [Semaphore](./02-semaphore.md) - 리소스 카운팅
- [Atomic Operations](./04-atomic-operations.md) - Lock-Free 대안
- [동시성 패턴](../04-concurrency-patterns/README.md) - 실전 패턴

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams, Chapter 4
- [cppreference: condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)
- [POSIX Threads: pthread_cond](https://man7.org/linux/man-pages/man3/pthread_cond_wait.3p.html)

---

*Condition Variable은 Busy-Waiting을 피하고 효율적인 스레드 협력을 가능하게 합니다. Predicate와 함께 사용하세요!*
