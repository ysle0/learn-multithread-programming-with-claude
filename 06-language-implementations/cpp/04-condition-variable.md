# Condition Variables in C++

Condition variables allow threads to wait for certain conditions to become true, enabling efficient communication and synchronization between threads without busy-waiting.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [std::condition_variable](#stdcondition_variable)
- [Wait Operations](#wait-operations)
- [Notify Operations](#notify-operations)
- [Common Patterns](#common-patterns)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is a Condition Variable?

A condition variable allows threads to:
1. **Wait** for a condition to become true (blocks thread)
2. **Signal** when condition changes (wakes waiting threads)

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_for_signal() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });  // Wait until ready is true
    std::cout << "Proceeding!\n";
}

void send_signal() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // Wake one waiting thread
}

int main() {
    std::thread t1(wait_for_signal);
    std::thread t2(send_signal);
    t1.join();
    t2.join();
    return 0;
}
```

### Why Use Condition Variables?

**Without CV (Busy Waiting - BAD):**
```cpp
// BAD: Wastes CPU
while (!ready) {
    std::this_thread::yield();  // Still busy-waiting
}
```

**With CV (Efficient - GOOD):**
```cpp
// GOOD: Thread sleeps until notified
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

## std::condition_variable

### Basic Structure

```cpp
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool condition = false;

void waiter() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return condition; });
    // Condition is now true, lock is held
}

void notifier() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        condition = true;
    }  // Release lock before notify
    cv.notify_one();
}
```

### Requirements

1. Must use `std::unique_lock` (not `std::lock_guard`)
2. Must protect condition with mutex
3. Must use predicate to avoid spurious wakeups

## Wait Operations

### wait() with Predicate

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
int data = 0;
bool ready = false;

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);

    // Wait until ready is true
    cv.wait(lock, [] { return ready; });

    // Now we can use data safely
    std::cout << "Data: " << data << "\n";
}

void producer() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        data = 42;
        ready = true;
    }
    cv.notify_one();
}

int main() {
    std::thread c(consumer);
    std::thread p(producer);
    c.join();
    p.join();
    return 0;
}
```

### wait() without Predicate (Manual Loop)

```cpp
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_manual() {
    std::unique_lock<std::mutex> lock(mtx);

    // Must loop to handle spurious wakeups
    while (!ready) {
        cv.wait(lock);  // Releases lock and sleeps
    }                   // Reacquires lock when woken

    // ready is now true
}
```

### wait_for() - Timed Wait

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <chrono>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_with_timeout() {
    std::unique_lock<std::mutex> lock(mtx);

    if (cv.wait_for(lock, std::chrono::seconds(1), [] { return ready; })) {
        std::cout << "Condition met!\n";
    } else {
        std::cout << "Timeout!\n";
    }
}

int main() {
    std::thread t(wait_with_timeout);
    // Don't signal, let it timeout
    t.join();
    return 0;
}
```

### wait_until() - Wait Until Time Point

```cpp
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_until_deadline() {
    std::unique_lock<std::mutex> lock(mtx);

    auto deadline = std::chrono::system_clock::now()
                  + std::chrono::seconds(2);

    if (cv.wait_until(lock, deadline, [] { return ready; })) {
        std::cout << "Condition met before deadline!\n";
    } else {
        std::cout << "Deadline passed!\n";
    }
}
```

## Notify Operations

### notify_one() - Wake One Thread

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void waiter(int id) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
    std::cout << "Thread " << id << " woken\n";
}

int main() {
    std::thread t1(waiter, 1);
    std::thread t2(waiter, 2);

    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }

    cv.notify_one();  // Only one thread wakes
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    cv.notify_one();  // Wake the second thread

    t1.join();
    t2.join();
    return 0;
}
```

### notify_all() - Wake All Threads

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <vector>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void waiter(int id) {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
    std::cout << "Thread " << id << " woken\n";
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(waiter, i);
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }

    cv.notify_all();  // Wake all waiting threads

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### When to Use notify_one vs. notify_all

```cpp
// Use notify_one when:
// - Only one thread should process the event
// - Example: Work queue with multiple workers

cv.notify_one();  // Wake one worker

// Use notify_all when:
// - All threads should respond to the event
// - Example: Barrier synchronization

cv.notify_all();  // Wake all threads
```

## Common Patterns

### Producer-Consumer Queue

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <iostream>

template<typename T>
class BlockingQueue {
    std::mutex mtx;
    std::condition_variable cv;
    std::queue<T> queue;
    bool done = false;

public:
    void push(T value) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            queue.push(std::move(value));
        }
        cv.notify_one();
    }

    bool pop(T& value) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this] { return !queue.empty() || done; });

        if (queue.empty()) {
            return false;  // Queue is done
        }

        value = std::move(queue.front());
        queue.pop();
        return true;
    }

    void finish() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            done = true;
        }
        cv.notify_all();
    }
};

int main() {
    BlockingQueue<int> queue;

    // Producer
    std::thread producer([&queue] {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        queue.finish();
    });

    // Consumer
    std::thread consumer([&queue] {
        int value;
        while (queue.pop(value)) {
            std::cout << "Consumed: " << value << "\n";
        }
    });

    producer.join();
    consumer.join();
    return 0;
}
```

### Barrier Synchronization

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <vector>

class Barrier {
    std::mutex mtx;
    std::condition_variable cv;
    size_t count;
    size_t const threshold;
    size_t generation = 0;

public:
    explicit Barrier(size_t count) : threshold(count), count(count) {}

    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        size_t gen = generation;

        if (--count == 0) {
            ++generation;
            count = threshold;
            cv.notify_all();
        } else {
            cv.wait(lock, [this, gen] { return gen != generation; });
        }
    }
};

void worker(int id, Barrier& barrier) {
    std::cout << "Thread " << id << " phase 1\n";
    barrier.wait();  // Synchronize

    std::cout << "Thread " << id << " phase 2\n";
    barrier.wait();  // Synchronize again

    std::cout << "Thread " << id << " done\n";
}

int main() {
    const int num_threads = 4;
    Barrier barrier(num_threads);

    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(worker, i, std::ref(barrier));
    }

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### Thread Pool with Condition Variable

```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>
#include <functional>
#include <vector>
#include <iostream>

class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop = false;

public:
    ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(mtx);
                        cv.wait(lock, [this] {
                            return stop || !tasks.empty();
                        });

                        if (stop && tasks.empty()) {
                            return;
                        }

                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
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
        cv.notify_one();
    }
};

int main() {
    ThreadPool pool(4);

    for (int i = 0; i < 10; ++i) {
        pool.enqueue([i] {
            std::cout << "Task " << i << " executing\n";
        });
    }

    std::this_thread::sleep_for(std::chrono::seconds(1));
    return 0;
}
```

### Event Signaling

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

class Event {
    std::mutex mtx;
    std::condition_variable cv;
    bool signaled = false;

public:
    void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this] { return signaled; });
    }

    void signal() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            signaled = true;
        }
        cv.notify_all();
    }

    void reset() {
        std::lock_guard<std::mutex> lock(mtx);
        signaled = false;
    }
};

int main() {
    Event event;

    std::thread worker([&event] {
        std::cout << "Worker waiting...\n";
        event.wait();
        std::cout << "Worker proceeding!\n";
    });

    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Signaling event\n";
    event.signal();

    worker.join();
    return 0;
}
```

## Comparison with Other Languages

### C++ vs. C#
```cpp
// C++
std::mutex mtx;
std::condition_variable cv;
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });

// C# equivalent:
// object lockObj = new object();
// lock (lockObj) {
//     while (!ready) {
//         Monitor.Wait(lockObj);
//     }
// }
```

### C++ vs. Go
```cpp
// C++ - Condition variable
std::condition_variable cv;
cv.wait(lock, [] { return ready; });

// Go - Channels (different paradigm)
// <-readyChan
```

### C++ vs. JavaScript
```cpp
// C++ has condition variables
std::condition_variable cv;

// JavaScript doesn't have condition variables
// (single-threaded main execution)
// Use Promises for async coordination
```

## Best Practices

### 1. Always Use a Predicate

```cpp
// BAD: Spurious wakeup not handled
cv.wait(lock);
// May wake up even if condition isn't true!

// GOOD: Predicate handles spurious wakeups
cv.wait(lock, [] { return ready; });
```

### 2. Protect Condition with Mutex

```cpp
// BAD: Condition not protected
ready = true;
cv.notify_one();  // Race condition!

// GOOD: Condition protected by mutex
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}
cv.notify_one();
```

### 3. Notify Outside Lock (When Possible)

```cpp
// GOOD: Notify after releasing lock
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}  // Lock released
cv.notify_one();  // Then notify

// ALSO OK: Notify inside lock (but less efficient)
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
    cv.notify_one();
}
```

### 4. Use unique_lock, Not lock_guard

```cpp
// REQUIRED: cv.wait needs unique_lock
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });

// WON'T COMPILE: lock_guard doesn't support unlock/lock
// std::lock_guard<std::mutex> lock(mtx);
// cv.wait(lock, [] { return ready; });  // ERROR
```

### 5. Avoid Busy-Waiting

```cpp
// BAD: Busy-waiting
while (!ready) {
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
}

// GOOD: Use condition variable
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

## Common Pitfalls

### 1. Spurious Wakeups

```cpp
// BAD: Assumes wakeup means condition is true
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);  // Might wake spuriously!
    // ready might still be false
}

// GOOD: Check condition in loop/predicate
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
    // ready is guaranteed true
}
```

### 2. Lost Wakeup

```cpp
// BAD: Notify before wait
void thread1() {
    ready = true;
    cv.notify_one();  // Sent before anyone waiting!
}

void thread2() {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });  // Might have missed signal
}

// GOOD: Use proper synchronization
// The predicate (ready) ensures correctness even if notify happens first
```

### 3. Deadlock with notify_one

```cpp
// SUBTLE: All threads might be waiting for different conditions
std::condition_variable cv;
bool cond1 = false, cond2 = false;

void thread1() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return cond1; });
}

void thread2() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return cond2; });
}

void notifier() {
    cond1 = true;
    cv.notify_one();  // Might wake thread2, which goes back to sleep!
}

// SOLUTION: Use notify_all or separate condition variables
```

### 4. Not Holding Lock When Checking Condition

```cpp
// BAD: Race condition
if (ready) {  // Checked without lock!
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
}

// GOOD: Check condition with lock held
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
}
```

### 5. Exception Safety

```cpp
// BAD: Lock not released if exception thrown
std::unique_lock<std::mutex> lock(mtx);
might_throw();  // Lock held if this throws!
cv.wait(lock, [] { return ready; });

// GOOD: RAII ensures lock released
{
    std::unique_lock<std::mutex> lock(mtx);
    try {
        might_throw();
    } catch (...) {
        // Lock automatically released
        throw;
    }
    cv.wait(lock, [] { return ready; });
}

// BETTER: Just rely on RAII
{
    std::unique_lock<std::mutex> lock(mtx);
    might_throw();  // Lock released on exception
    cv.wait(lock, [] { return ready; });
}
```

## Internal Mechanisms

### Futex-Based Implementation (Linux)

`std::condition_variable`은 내부적으로 pthread_cond_t를 사용하며, 이는 futex 기반입니다:

```cpp
// pthread_cond_t 내부 구조 (glibc 단순화)
struct __pthread_cond_s {
    __atomic_uint64_t __wseq;    // waiter sequence (다음 대기자 번호)
    __atomic_uint64_t __g1_start; // group 1 시작 시퀀스
    unsigned int __g_refs[2];     // 그룹별 참조 카운트
    unsigned int __g_size[2];     // 그룹별 대기자 수
    unsigned int __g1_orig_size;  // 원래 그룹 크기
    unsigned int __wrefs;         // writer 참조
    unsigned int __g_signals[2];  // 그룹별 시그널 카운트
};
```

### wait() 내부 동작 흐름

```
cv.wait(lock, pred)
    │
    ▼
while (!pred())  ← spurious wakeup 처리
    │
    ▼
┌─ 1. lock.unlock()  ← 뮤텍스 해제
│
├─ 2. __wseq 증가 (대기 순번 획득)
│
├─ 3. futex(&__g_signals[g], FUTEX_WAIT, ...)
│      │
│      └─ 커널: wait queue에 추가, sleep
│
├─ (notify로 깨어남)
│
├─ 4. __g_refs 감소
│
└─ 5. lock.lock()  ← 뮤텍스 재획득
```

**Spurious Wakeup 발생 원인**:
1. **Futex 구현**: 리눅스 커널이 프로세스를 임의로 깨울 수 있음
2. **Broadcast 최적화**: notify_all 시 여러 스레드가 깨지만 조건은 하나만 충족
3. **그룹 전환**: 내부 그룹 관리 시 추가 wakeup 발생 가능

```cpp
// 반드시 루프 또는 predicate 사용
cv.wait(lock, []{ return ready; });

// 내부적으로 다음과 같음:
while (!ready) {
    cv.wait(lock);  // spurious wakeup 가능
}
```

### notify_one vs notify_all 커널 동작

```cpp
// notify_one: 하나의 대기자만 깨움
void notify_one() {
    // 1. __g_signals[g] 증가
    // 2. futex(&__g_signals[g], FUTEX_WAKE, 1)
    //    - 커널: wait queue에서 하나만 깨움
}

// notify_all: 모든 대기자 깨움
void notify_all() {
    // 1. 현재 그룹의 모든 대기자에게 시그널
    // 2. futex(&__g_signals[g], FUTEX_WAKE, INT_MAX)
    // 3. 그룹 전환 (새 대기자는 새 그룹으로)
}
```

**Thundering Herd 문제**:
```
notify_all() 호출
    │
    ▼
모든 대기자 깨어남 (N개)
    │
    ▼
뮤텍스 획득 경쟁
    │
    ├─ 1개 스레드: 뮤텍스 획득, 작업 수행
    │
    └─ N-1개 스레드: 뮤텍스 대기 → 조건 확인 → 다시 sleep
         (CPU 낭비!)

해결책:
- notify_one 사용 (적절한 경우)
- 조건을 더 세분화 (여러 condition_variable)
```

### condition_variable_any 구현

`std::condition_variable`은 `std::unique_lock<std::mutex>`만 지원하지만,
`std::condition_variable_any`는 모든 Lockable 타입 지원:

```cpp
// condition_variable_any 내부 구조
class condition_variable_any {
    condition_variable _M_cond;
    shared_ptr<mutex> _M_mutex;  // 내부 뮤텍스

public:
    template<typename _Lock>
    void wait(_Lock& __lock) {
        // 1. 내부 뮤텍스로 상태 보호
        unique_lock<mutex> __my_lock(*_M_mutex);

        // 2. 외부 락 해제
        __lock.unlock();

        // 3. 내부 조건변수 대기
        _M_cond.wait(__my_lock);

        // 4. 외부 락 재획득
        __lock.lock();
    }
};

// 추가 오버헤드: 내부 뮤텍스 잠금/해제
// 가능하면 condition_variable + unique_lock<mutex> 사용
```

### Windows Condition Variable 구현

```cpp
// Windows CONDITION_VARIABLE
typedef struct _RTL_CONDITION_VARIABLE {
    PVOID Ptr;  // SRWLock 스타일의 내부 포인터
} CONDITION_VARIABLE;

// wait는 내부적으로 NtWaitForKeyedEvent 사용
void SleepConditionVariableCS(cv, cs, timeout) {
    // 1. Critical Section 해제
    // 2. Keyed Event 대기
    // 3. Critical Section 재획득
}
```

### Wait Morphing 최적화

일부 구현에서 notify_one이 대기자를 조건변수 큐에서 뮤텍스 큐로 직접 이동:

```
일반 구현:
notify_one() → waiter 깨움 → waiter가 mutex lock 시도

Wait Morphing:
notify_one() → waiter를 mutex wait queue로 이동
              (context switch 감소)
```

**Linux glibc의 FUTEX_REQUEUE**:
```cpp
// notify + 뮤텍스 전환을 원자적으로
futex(&cond->__g_signals, FUTEX_REQUEUE,
      1,                    // 깨울 개수
      INT_MAX,              // requeue할 개수
      &mutex->__lock,       // 목적지 futex
      0);
```

## Performance Considerations

### Cost of Operations

```cpp
// cv.wait() - Relatively expensive
// - Mutex unlock
// - Thread context switch (to sleep)
// - Thread context switch (when woken)
// - Mutex lock

// Use wisely:
// - GOOD for infrequent events
// - BAD for high-frequency signaling (consider atomics instead)
```

### Thundering Herd Problem

```cpp
// PROBLEM: notify_all wakes many threads, but only one can proceed
std::condition_variable cv;
bool work_available = false;

// Many workers:
cv.wait(lock, [] { return work_available; });
// All wake up, but only one gets work

// SOLUTION: Use notify_one for work queue
cv.notify_one();  // Wake only one worker
```

## Complete Example: Semaphore Implementation

```cpp
#include <mutex>
#include <condition_variable>
#include <iostream>
#include <thread>
#include <vector>

class Semaphore {
    std::mutex mtx;
    std::condition_variable cv;
    int count;

public:
    explicit Semaphore(int count) : count(count) {}

    void acquire() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this] { return count > 0; });
        --count;
    }

    void release() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            ++count;
        }
        cv.notify_one();
    }
};

Semaphore sem(2);  // Max 2 concurrent accesses

void worker(int id) {
    sem.acquire();
    std::cout << "Worker " << id << " entering\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Worker " << id << " leaving\n";
    sem.release();
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back(worker, i);
    }

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

## Further Reading

- [C++ Reference: std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)
- [Mutex and Lock Guard](./02-mutex-lock-guard.md)
- [Async and Future](./05-async-future.md)

## Navigation

- [Back to C++ Overview](./README.md)
- Previous: [Atomic Operations](./03-atomic.md)
- Next: [Async and Future](./05-async-future.md)
