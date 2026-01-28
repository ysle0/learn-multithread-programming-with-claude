# Mutex and Lock Guards in C++

Mutexes (mutual exclusion) are the fundamental synchronization primitive for protecting shared data from concurrent access. C++ provides RAII-based lock guards to ensure exception-safe locking.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [std::mutex](#stdmutex)
- [Lock Guards](#lock-guards)
- [Unique Lock](#unique-lock)
- [Shared Mutex](#shared-mutex)
- [Other Mutex Types](#other-mutex-types)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is a Mutex?

A mutex ensures that only one thread can access a protected resource at a time:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int counter = 0;

void increment() {
    mtx.lock();
    ++counter;  // Protected by mutex
    mtx.unlock();
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n";  // Always 2
    return 0;
}
```

### RAII Principle

Never lock/unlock manually! Use RAII wrappers:
```cpp
// BAD: Manual locking
mtx.lock();
do_work();  // What if this throws?
mtx.unlock();

// GOOD: RAII lock guard
{
    std::lock_guard<std::mutex> lock(mtx);
    do_work();  // Lock released even if exception thrown
}
```

## std::mutex

### Basic Usage
```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

class Counter {
    std::mutex mtx;
    int value = 0;

public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);
        ++value;
    }

    int get() {
        std::lock_guard<std::mutex> lock(mtx);
        return value;
    }
};

int main() {
    Counter counter;
    std::vector<std::thread> threads;

    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < 1000; ++j) {
                counter.increment();
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Final value: " << counter.get() << "\n";  // 10000
    return 0;
}
```

### Manual Lock/Unlock (Not Recommended)
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

void manual_locking() {
    mtx.lock();
    try {
        std::cout << "Critical section\n";
        // Do work
        mtx.unlock();
    } catch (...) {
        mtx.unlock();  // Must unlock in exception handler too!
        throw;
    }
}
```

### Try Lock
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;

void try_lock_example() {
    if (mtx.try_lock()) {
        std::cout << "Lock acquired\n";
        // Do work
        mtx.unlock();
    } else {
        std::cout << "Lock not available, doing other work\n";
    }
}

int main() {
    std::thread t1(try_lock_example);
    std::thread t2(try_lock_example);
    t1.join();
    t2.join();
    return 0;
}
```

## Lock Guards

### std::lock_guard (C++11)

Simplest RAII lock - acquires on construction, releases on destruction:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;

void safe_print(const std::string& msg) {
    std::lock_guard<std::mutex> lock(mtx);
    std::cout << msg << "\n";
}  // Lock automatically released

int main() {
    std::thread t1(safe_print, "Thread 1");
    std::thread t2(safe_print, "Thread 2");
    t1.join();
    t2.join();
    return 0;
}
```

### std::scoped_lock (C++17)

Can lock multiple mutexes atomically (prevents deadlock):
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx1, mtx2;
int resource1 = 0, resource2 = 0;

void transfer_v1() {
    // BAD: Can deadlock
    std::lock_guard<std::mutex> lock1(mtx1);
    std::lock_guard<std::mutex> lock2(mtx2);
    ++resource1;
    --resource2;
}

void transfer_v2() {
    // GOOD: Atomic locking, no deadlock
    std::scoped_lock lock(mtx1, mtx2);
    ++resource1;
    --resource2;
}

int main() {
    std::thread t1(transfer_v2);
    std::thread t2(transfer_v2);
    t1.join();
    t2.join();
    std::cout << "Resource1: " << resource1 << ", Resource2: "
              << resource2 << "\n";
    return 0;
}
```

### Locking Multiple Mutexes
```cpp
#include <mutex>
#include <thread>

class BankAccount {
    std::mutex mtx;
    double balance;

public:
    BankAccount(double initial) : balance(initial) {}

    friend void transfer(BankAccount& from, BankAccount& to, double amount) {
        // Lock both mutexes without deadlock
        std::scoped_lock lock(from.mtx, to.mtx);
        from.balance -= amount;
        to.balance += amount;
    }

    double get_balance() {
        std::lock_guard<std::mutex> lock(mtx);
        return balance;
    }
};

int main() {
    BankAccount alice(1000);
    BankAccount bob(500);

    std::thread t1([&] { transfer(alice, bob, 100); });
    std::thread t2([&] { transfer(bob, alice, 50); });

    t1.join();
    t2.join();
    return 0;
}
```

## Unique Lock

### std::unique_lock (C++11)

More flexible than `lock_guard`, allows deferred locking, try-lock, timed locks:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;

void deferred_lock_example() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // Mutex not locked yet

    // Do some work without lock
    std::cout << "Work without lock\n";

    // Now lock
    lock.lock();
    std::cout << "Work with lock\n";
    lock.unlock();

    // Can lock again
    lock.lock();
    std::cout << "More work with lock\n";
}  // Automatically unlocked if still locked

int main() {
    std::thread t(deferred_lock_example);
    t.join();
    return 0;
}
```

### Timed Locking
```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

std::timed_mutex tmtx;

void try_lock_for_example() {
    std::unique_lock<std::timed_mutex> lock(tmtx, std::defer_lock);

    if (lock.try_lock_for(std::chrono::milliseconds(100))) {
        std::cout << "Lock acquired within 100ms\n";
        // Do work
    } else {
        std::cout << "Timeout: couldn't acquire lock\n";
    }
}

int main() {
    std::thread t1(try_lock_for_example);
    std::thread t2(try_lock_for_example);
    t1.join();
    t2.join();
    return 0;
}
```

### Manual Lock/Unlock with Unique Lock
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

void flexible_locking() {
    std::unique_lock<std::mutex> lock(mtx);

    std::cout << "Locked\n";
    // Do some work

    lock.unlock();
    std::cout << "Unlocked, doing other work\n";
    // Do work without lock

    lock.lock();
    std::cout << "Locked again\n";
    // More work with lock
}
```

### Moving Unique Locks
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

std::unique_lock<std::mutex> get_lock() {
    std::unique_lock<std::mutex> lock(mtx);
    return lock;  // Move semantics
}

void use_lock() {
    auto lock = get_lock();
    std::cout << "Have lock from function\n";
}
```

### Condition Variable Compatibility
```cpp
#include <mutex>
#include <condition_variable>
#include <queue>
#include <thread>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

void consumer() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return !queue.empty(); });
    // Process queue
}
// Note: condition_variable requires unique_lock, not lock_guard
```

## Shared Mutex

### std::shared_mutex (C++17)

Reader-writer lock: multiple readers OR one writer:
```cpp
#include <shared_mutex>
#include <thread>
#include <iostream>
#include <vector>

class ThreadSafeCounter {
    mutable std::shared_mutex mtx;
    int value = 0;

public:
    // Multiple readers can call this simultaneously
    int read() const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        return value;
    }

    // Only one writer allowed
    void increment() {
        std::unique_lock<std::shared_mutex> lock(mtx);
        ++value;
    }

    void write(int v) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        value = v;
    }
};

int main() {
    ThreadSafeCounter counter;
    std::vector<std::thread> threads;

    // Many readers
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < 100; ++j) {
                std::cout << counter.read() << " ";
            }
        });
    }

    // Few writers
    for (int i = 0; i < 2; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < 50; ++j) {
                counter.increment();
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "\nFinal: " << counter.read() << "\n";
    return 0;
}
```

### Read-Write Lock Example
```cpp
#include <shared_mutex>
#include <map>
#include <string>
#include <thread>

class ThreadSafeMap {
    mutable std::shared_mutex mtx;
    std::map<std::string, int> data;

public:
    // Read operation - many threads can read simultaneously
    int get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        auto it = data.find(key);
        return it != data.end() ? it->second : 0;
    }

    // Write operation - exclusive access
    void set(const std::string& key, int value) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        data[key] = value;
    }

    // Write operation - exclusive access
    void remove(const std::string& key) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        data.erase(key);
    }
};
```

## Other Mutex Types

### std::recursive_mutex

Allows same thread to lock multiple times:
```cpp
#include <mutex>
#include <iostream>

class RecursiveCounter {
    std::recursive_mutex mtx;
    int value = 0;

    void increment_internal() {
        std::lock_guard<std::recursive_mutex> lock(mtx);
        ++value;
    }

public:
    void increment() {
        std::lock_guard<std::recursive_mutex> lock(mtx);
        increment_internal();  // Same thread locks again - OK
    }

    int get() {
        std::lock_guard<std::recursive_mutex> lock(mtx);
        return value;
    }
};

int main() {
    RecursiveCounter counter;
    counter.increment();
    std::cout << counter.get() << "\n";
    return 0;
}
```

### std::timed_mutex

Supports timeout operations:
```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

std::timed_mutex tmtx;

void try_lock_example() {
    using namespace std::chrono_literals;

    if (tmtx.try_lock_for(100ms)) {
        std::cout << "Lock acquired\n";
        std::this_thread::sleep_for(200ms);
        tmtx.unlock();
    } else {
        std::cout << "Timeout\n";
    }
}

int main() {
    std::thread t1(try_lock_example);
    std::thread t2(try_lock_example);
    t1.join();
    t2.join();
    return 0;
}
```

## Comparison with Other Languages

### C++ vs. C#
```cpp
// C++
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);
    // Critical section
}

// C# equivalent:
// private object lockObj = new object();
// lock (lockObj) {
//     // Critical section
// }
```

### C++ vs. Go
```cpp
// C++
std::mutex mtx;
mtx.lock();
// Critical section
mtx.unlock();

// Go equivalent:
// var mu sync.Mutex
// mu.Lock()
// // Critical section
// mu.Unlock()
```

### C++ vs. JavaScript
```cpp
// C++ has real mutexes
std::mutex mtx;

// JavaScript has no equivalent (single-threaded main execution)
// For Workers, use Atomics.wait/notify with SharedArrayBuffer
```

## Best Practices

### 1. Always Use RAII Guards
```cpp
// GOOD: Automatic unlock
{
    std::lock_guard<std::mutex> lock(mtx);
    critical_section();
}

// BAD: Manual unlock
mtx.lock();
critical_section();
mtx.unlock();
```

### 2. Keep Critical Sections Small
```cpp
// BAD: Long critical section
{
    std::lock_guard<std::mutex> lock(mtx);
    expensive_computation();  // Don't hold lock during this!
    shared_data = result;
}

// GOOD: Minimal critical section
auto result = expensive_computation();
{
    std::lock_guard<std::mutex> lock(mtx);
    shared_data = result;
}
```

### 3. Lock Ordering to Prevent Deadlock
```cpp
// GOOD: Always lock in same order
void transfer(Account& from, Account& to, double amount) {
    // Lock lower address first
    std::mutex* first = &from.mtx < &to.mtx ? &from.mtx : &to.mtx;
    std::mutex* second = &from.mtx < &to.mtx ? &to.mtx : &from.mtx;

    std::lock_guard<std::mutex> lock1(*first);
    std::lock_guard<std::mutex> lock2(*second);

    from.balance -= amount;
    to.balance += amount;
}

// BETTER: Use scoped_lock
void transfer(Account& from, Account& to, double amount) {
    std::scoped_lock lock(from.mtx, to.mtx);
    from.balance -= amount;
    to.balance += amount;
}
```

### 4. Use Shared Mutex for Read-Heavy Workloads
```cpp
class Cache {
    mutable std::shared_mutex mtx;
    std::map<std::string, std::string> data;

public:
    // Many readers - use shared lock
    std::string get(const std::string& key) const {
        std::shared_lock lock(mtx);
        return data.at(key);
    }

    // Few writers - use unique lock
    void set(const std::string& key, const std::string& value) {
        std::unique_lock lock(mtx);
        data[key] = value;
    }
};
```

### 5. Avoid Recursive Mutexes When Possible
```cpp
// BAD: Using recursive mutex to paper over design issues
class BadDesign {
    std::recursive_mutex mtx;

    void foo() {
        std::lock_guard lock(mtx);
        bar();  // Locks again
    }

    void bar() {
        std::lock_guard lock(mtx);
        // Work
    }
};

// GOOD: Refactor to avoid recursion
class GoodDesign {
    std::mutex mtx;

    void bar_internal() {
        // Work (assumes lock held)
    }

public:
    void foo() {
        std::lock_guard lock(mtx);
        bar_internal();
    }

    void bar() {
        std::lock_guard lock(mtx);
        bar_internal();
    }
};
```

## Common Pitfalls

### 1. Forgetting to Lock
```cpp
// BAD: No synchronization
class UnsafeCounter {
    int value = 0;
public:
    void increment() { ++value; }  // Race condition!
};

// GOOD
class SafeCounter {
    std::mutex mtx;
    int value = 0;
public:
    void increment() {
        std::lock_guard lock(mtx);
        ++value;
    }
};
```

### 2. Deadlock with Multiple Locks
```cpp
// BAD: Can deadlock
std::mutex m1, m2;

void thread1() {
    std::lock_guard lock1(m1);
    std::lock_guard lock2(m2);
}

void thread2() {
    std::lock_guard lock2(m2);  // Reversed order!
    std::lock_guard lock1(m1);
}

// GOOD: Use scoped_lock
void thread1() {
    std::scoped_lock lock(m1, m2);
}

void thread2() {
    std::scoped_lock lock(m1, m2);  // Order doesn't matter
}
```

### 3. Locking Too Much
```cpp
// BAD: Holding lock while doing I/O
{
    std::lock_guard lock(mtx);
    std::cout << shared_data << "\n";  // I/O with lock!
}

// GOOD: Copy data, release lock, then do I/O
std::string data_copy;
{
    std::lock_guard lock(mtx);
    data_copy = shared_data;
}
std::cout << data_copy << "\n";
```

### 4. Not Protecting All Access
```cpp
// BAD: Inconsistent protection
class BadCache {
    std::mutex mtx;
    std::map<int, int> data;

public:
    void set(int key, int value) {
        std::lock_guard lock(mtx);
        data[key] = value;
    }

    int get(int key) {
        return data[key];  // Forgot to lock!
    }
};
```

### 5. Returning References to Protected Data
```cpp
// BAD: Exposes protected data
class BadContainer {
    std::mutex mtx;
    std::vector<int> data;

public:
    std::vector<int>& get_data() {
        std::lock_guard lock(mtx);
        return data;  // Lock released but reference escapes!
    }
};

// GOOD: Return a copy
class GoodContainer {
    std::mutex mtx;
    std::vector<int> data;

public:
    std::vector<int> get_data() {
        std::lock_guard lock(mtx);
        return data;  // Copy
    }
};
```

## Internal Mechanisms

### std::mutex Futex Implementation (Linux/glibc)

```cpp
// pthread_mutex 내부 구조 (glibc NPTL)
struct __pthread_mutex_s {
    int __lock;           // 0: unlocked, 1: locked, 2: contended
    unsigned int __count; // recursive lock count
    int __owner;          // owning thread ID (for recursive/errorcheck)
    // ... additional fields for robustness, priority, etc.
};
```

**Lock 동작 (Fast Path + Slow Path)**:
```
lock() 호출
    │
    ▼
atomic_cmpxchg(&__lock, 0, 1)  ← Fast Path (user-space)
    │
    ├─ 성공 (0→1): 락 획득 완료, return
    │
    └─ 실패 (이미 locked)
         │
         ▼
    __lock을 2로 설정 (contended 표시)
         │
         ▼
    futex(&__lock, FUTEX_WAIT, 2)  ← Slow Path (커널 진입)
         │
         ▼
    커널 wait queue에서 sleep
         │
    (unlock 시 FUTEX_WAKE로 깨어남)
         │
         ▼
    재시도 루프
```

**Unlock 동작**:
```cpp
void unlock() {
    int old = atomic_exchange(&__lock, 0);  // 락 해제

    if (old == 2) {  // contended 상태였으면
        futex(&__lock, FUTEX_WAKE, 1);  // 대기자 1명 깨움
    }
}
```

### Windows CRITICAL_SECTION 내부 구조

```cpp
typedef struct _RTL_CRITICAL_SECTION {
    PRTL_CRITICAL_SECTION_DEBUG DebugInfo;  // 디버깅 정보
    LONG LockCount;                          // -1: unlocked, 0+: locked
    LONG RecursionCount;                     // 재귀 잠금 횟수
    HANDLE OwningThread;                     // 소유 스레드 ID
    HANDLE LockSemaphore;                    // 대기용 커널 세마포어
    ULONG_PTR SpinCount;                     // 스핀 횟수 (기본 4000)
} RTL_CRITICAL_SECTION;
```

**Spin 최적화**:
```cpp
void EnterCriticalSection(cs) {
    // 1단계: SpinCount 동안 busy-wait
    for (int i = 0; i < cs->SpinCount; i++) {
        if (TryEnterCriticalSection(cs))
            return;
        YieldProcessor();  // PAUSE instruction
    }

    // 2단계: 커널 세마포어 대기
    WaitForSingleObject(cs->LockSemaphore, INFINITE);
}
```

### lock_guard vs unique_lock 구현

```cpp
// std::lock_guard - 최소한의 RAII 래퍼
template<typename _Mutex>
class lock_guard {
    _Mutex& _M_device;

public:
    explicit lock_guard(_Mutex& __m) : _M_device(__m) {
        _M_device.lock();  // 생성 시 락
    }

    ~lock_guard() {
        _M_device.unlock();  // 소멸 시 언락
    }

    // 복사/이동 금지
    lock_guard(const lock_guard&) = delete;
    lock_guard& operator=(const lock_guard&) = delete;
};

// std::unique_lock - 더 유연한 RAII 래퍼
template<typename _Mutex>
class unique_lock {
    _Mutex* _M_device;    // 포인터 (null 가능)
    bool _M_owns;         // 락 소유 여부

public:
    // 지연 락
    unique_lock(_Mutex& __m, defer_lock_t) noexcept
        : _M_device(&__m), _M_owns(false) {}

    // 조건변수 호환: lock/unlock 수동 호출 가능
    void lock() {
        _M_device->lock();
        _M_owns = true;
    }

    void unlock() {
        _M_device->unlock();
        _M_owns = false;
    }

    ~unique_lock() {
        if (_M_owns)
            _M_device->unlock();
    }
};
```

### scoped_lock Deadlock Avoidance Algorithm

`std::scoped_lock`은 여러 뮤텍스를 데드락 없이 잠급니다:

```cpp
// std::lock 알고리즘 (try-and-back-off)
template<typename _L1, typename _L2, typename... _L3>
void lock(_L1& __l1, _L2& __l2, _L3&... __l3) {
    while (true) {
        // 첫 번째 락 획득
        unique_lock<_L1> __first(__l1);

        // 나머지 락들 try_lock 시도
        int __idx = __try_lock(__l2, __l3...);

        if (__idx == -1) {  // 모두 성공
            __first.release();  // RAII 해제 (락은 유지)
            return;
        }

        // 실패: 첫 번째 락 해제 후 재시도
        // (실패한 락이 다음 번 첫 번째가 되도록 순환)
    }
}
```

**Try-and-Back-Off 동작 예시**:
```
Thread 1: lock(A, B)          Thread 2: lock(B, A)
    │                              │
    ▼                              ▼
lock(A) ✓                     lock(B) ✓
try_lock(B) ✗ (T2 보유)      try_lock(A) ✗ (T1 보유)
unlock(A)                     unlock(B)
    │                              │
    ▼                              ▼
lock(B) 시도...               lock(A) 시도...
(순환하며 재시도, 결국 한 쪽이 성공)
```

### shared_mutex Reader-Writer Implementation

```cpp
// libstdc++ shared_mutex 상태
class shared_mutex {
    // 단일 atomic 값으로 상태 관리
    // 상위 비트: exclusive lock 여부
    // 하위 비트: reader count
    //
    // 0x00000000: free
    // 0x00000001: 1 reader
    // 0x80000000: 1 writer
    // 0x00000003: 3 readers

    unsigned int _M_state;

    void lock() {  // exclusive (writer)
        // 1. 상위 비트 설정 (writer 대기 표시)
        // 2. reader count가 0이 될 때까지 대기
        // 3. exclusive 획득
    }

    void lock_shared() {  // shared (reader)
        // writer가 없으면 reader count 증가
        // writer가 있거나 대기 중이면 대기
    }
};
```

### Recursive Mutex Counter Overflow

```cpp
// recursive_mutex의 재귀 횟수 제한
class recursive_mutex {
    unsigned int _M_count;  // 보통 32비트

    void lock() {
        if (/* 이미 소유 */) {
            if (_M_count == numeric_limits<unsigned int>::max())
                throw system_error(...);  // overflow!
            ++_M_count;
        } else {
            // 일반 락 획득
            _M_count = 1;
        }
    }
};
```

## Performance Considerations

### Lock Overhead
- **Uncontended lock**: ~25 nanoseconds
- **Contended lock**: Can be 1000x slower (microseconds)
- **Context switch**: 1-10 microseconds

### Lock Granularity
```cpp
// Fine-grained: More parallelism, more overhead
class FineGrained {
    std::mutex mtx1, mtx2;
    int data1, data2;

public:
    void update1(int v) {
        std::lock_guard lock(mtx1);
        data1 = v;
    }

    void update2(int v) {
        std::lock_guard lock(mtx2);
        data2 = v;
    }
};

// Coarse-grained: Less overhead, less parallelism
class CoarseGrained {
    std::mutex mtx;
    int data1, data2;

public:
    void update1(int v) {
        std::lock_guard lock(mtx);
        data1 = v;
    }

    void update2(int v) {
        std::lock_guard lock(mtx);
        data2 = v;
    }
};
```

## Complete Example: Thread-Safe Queue
```cpp
#include <mutex>
#include <queue>
#include <condition_variable>
#include <thread>
#include <iostream>

template<typename T>
class ThreadSafeQueue {
    mutable std::mutex mtx;
    std::queue<T> queue;
    std::condition_variable cv;

public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(std::move(value));
        cv.notify_one();
    }

    bool try_pop(T& value) {
        std::lock_guard<std::mutex> lock(mtx);
        if (queue.empty()) {
            return false;
        }
        value = std::move(queue.front());
        queue.pop();
        return true;
    }

    void wait_and_pop(T& value) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this] { return !queue.empty(); });
        value = std::move(queue.front());
        queue.pop();
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx);
        return queue.empty();
    }
};

int main() {
    ThreadSafeQueue<int> queue;

    // Producer
    std::thread producer([&queue] {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
    });

    // Consumer
    std::thread consumer([&queue] {
        for (int i = 0; i < 10; ++i) {
            int value;
            queue.wait_and_pop(value);
            std::cout << "Consumed: " << value << "\n";
        }
    });

    producer.join();
    consumer.join();
    return 0;
}
```

## Further Reading

- [C++ Reference: std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)
- [C++ Reference: std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)
- [Condition Variables](./04-condition-variable.md)

## Navigation

- [Back to C++ Overview](./README.md)
- Previous: [std::thread](./01-std-thread.md)
- Next: [Atomic Operations](./03-atomic.md)
