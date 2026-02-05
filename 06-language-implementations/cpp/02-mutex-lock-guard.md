# C++의 Mutex와 Lock Guard

Mutex(상호 배제)는 공유 데이터를 동시 접근으로부터 보호하기 위한 기본 동기화 프리미티브입니다. C++는 예외 안전한 잠금을 보장하는 RAII 기반 lock guard를 제공합니다.

## 목차
- [기본 개념](#기본-개념)
- [std::mutex](#stdmutex)
- [Lock Guard](#lock-guard)
- [Unique Lock](#unique-lock)
- [Shared Mutex](#shared-mutex)
- [기타 Mutex 타입](#기타-mutex-타입)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### Mutex란?

Mutex는 한 번에 하나의 스레드만 보호된 리소스에 접근할 수 있도록 보장합니다:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int counter = 0;

void increment() {
    mtx.lock();
    ++counter;  // mutex로 보호됨
    mtx.unlock();
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n";  // 항상 2
    return 0;
}
```

### RAII 원칙

절대 수동으로 lock/unlock 하지 마세요! RAII 래퍼를 사용하세요:
```cpp
// 나쁨: 수동 잠금
mtx.lock();
do_work();  // 예외가 발생하면?
mtx.unlock();

// 좋음: RAII lock guard
{
    std::lock_guard<std::mutex> lock(mtx);
    do_work();  // 예외가 발생해도 잠금 해제됨
}
```

## std::mutex

### 기본 사용법
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

### 수동 Lock/Unlock (권장하지 않음)
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

void manual_locking() {
    mtx.lock();
    try {
        std::cout << "Critical section\n";
        // 작업 수행
        mtx.unlock();
    } catch (...) {
        mtx.unlock();  // 예외 핸들러에서도 unlock 해야 함!
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
        // 작업 수행
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

## Lock Guard

### std::lock_guard (C++11)

가장 간단한 RAII 잠금 - 생성 시 획득, 소멸 시 해제:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;

void safe_print(const std::string& msg) {
    std::lock_guard<std::mutex> lock(mtx);
    std::cout << msg << "\n";
}  // 잠금 자동 해제

int main() {
    std::thread t1(safe_print, "Thread 1");
    std::thread t2(safe_print, "Thread 2");
    t1.join();
    t2.join();
    return 0;
}
```

### std::scoped_lock (C++17)

여러 mutex를 원자적으로 잠글 수 있습니다 (데드락 방지):
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx1, mtx2;
int resource1 = 0, resource2 = 0;

void transfer_v1() {
    // 나쁨: 데드락 발생 가능
    std::lock_guard<std::mutex> lock1(mtx1);
    std::lock_guard<std::mutex> lock2(mtx2);
    ++resource1;
    --resource2;
}

void transfer_v2() {
    // 좋음: 원자적 잠금, 데드락 없음
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

### 여러 Mutex 잠그기
```cpp
#include <mutex>
#include <thread>

class BankAccount {
    std::mutex mtx;
    double balance;

public:
    BankAccount(double initial) : balance(initial) {}

    friend void transfer(BankAccount& from, BankAccount& to, double amount) {
        // 데드락 없이 두 mutex 잠금
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

`lock_guard`보다 유연하며, 지연 잠금, try-lock, 시간 제한 잠금을 지원합니다:
```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;

void deferred_lock_example() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // 아직 mutex가 잠기지 않음

    // 잠금 없이 작업 수행
    std::cout << "Work without lock\n";

    // 이제 잠금
    lock.lock();
    std::cout << "Work with lock\n";
    lock.unlock();

    // 다시 잠금 가능
    lock.lock();
    std::cout << "More work with lock\n";
}  // 아직 잠겨있으면 자동으로 unlock

int main() {
    std::thread t(deferred_lock_example);
    t.join();
    return 0;
}
```

### 시간 제한 잠금
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
        // 작업 수행
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

### Unique Lock으로 수동 Lock/Unlock
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

void flexible_locking() {
    std::unique_lock<std::mutex> lock(mtx);

    std::cout << "Locked\n";
    // 작업 수행

    lock.unlock();
    std::cout << "Unlocked, doing other work\n";
    // 잠금 없이 작업 수행

    lock.lock();
    std::cout << "Locked again\n";
    // 잠금이 필요한 추가 작업
}
```

### Unique Lock 이동
```cpp
#include <mutex>
#include <iostream>

std::mutex mtx;

std::unique_lock<std::mutex> get_lock() {
    std::unique_lock<std::mutex> lock(mtx);
    return lock;  // 이동 의미론
}

void use_lock() {
    auto lock = get_lock();
    std::cout << "Have lock from function\n";
}
```

### Condition Variable 호환성
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
    // 큐 처리
}
// 참고: condition_variable은 lock_guard가 아닌 unique_lock이 필요
```

## Shared Mutex

### std::shared_mutex (C++17)

Reader-Writer 잠금: 여러 Reader 또는 하나의 Writer:
```cpp
#include <shared_mutex>
#include <thread>
#include <iostream>
#include <vector>

class ThreadSafeCounter {
    mutable std::shared_mutex mtx;
    int value = 0;

public:
    // 여러 reader가 동시에 호출 가능
    int read() const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        return value;
    }

    // writer는 하나만 허용
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

    // 다수의 reader
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < 100; ++j) {
                std::cout << counter.read() << " ";
            }
        });
    }

    // 소수의 writer
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

### Read-Write Lock 예제
```cpp
#include <shared_mutex>
#include <map>
#include <string>
#include <thread>

class ThreadSafeMap {
    mutable std::shared_mutex mtx;
    std::map<std::string, int> data;

public:
    // 읽기 연산 - 여러 스레드가 동시에 읽기 가능
    int get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mtx);
        auto it = data.find(key);
        return it != data.end() ? it->second : 0;
    }

    // 쓰기 연산 - 배타적 접근
    void set(const std::string& key, int value) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        data[key] = value;
    }

    // 쓰기 연산 - 배타적 접근
    void remove(const std::string& key) {
        std::unique_lock<std::shared_mutex> lock(mtx);
        data.erase(key);
    }
};
```

## 기타 Mutex 타입

### std::recursive_mutex

같은 스레드가 여러 번 잠글 수 있습니다:
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
        increment_internal();  // 같은 스레드가 다시 잠금 - OK
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

타임아웃 연산을 지원합니다:
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

## 다른 언어와의 비교

### C++ vs. C#
```cpp
// C++
std::mutex mtx;
{
    std::lock_guard<std::mutex> lock(mtx);
    // 임계 영역
}

// C# 동등 코드:
// private object lockObj = new object();
// lock (lockObj) {
//     // 임계 영역
// }
```

### C++ vs. Go
```cpp
// C++
std::mutex mtx;
mtx.lock();
// 임계 영역
mtx.unlock();

// Go 동등 코드:
// var mu sync.Mutex
// mu.Lock()
// // 임계 영역
// mu.Unlock()
```

### C++ vs. JavaScript
```cpp
// C++는 실제 mutex를 가짐
std::mutex mtx;

// JavaScript에는 동등한 것이 없음 (단일 스레드 메인 실행)
// Worker의 경우 SharedArrayBuffer와 함께 Atomics.wait/notify 사용
```

## 모범 사례

### 1. 항상 RAII Guard 사용
```cpp
// 좋음: 자동 unlock
{
    std::lock_guard<std::mutex> lock(mtx);
    critical_section();
}

// 나쁨: 수동 unlock
mtx.lock();
critical_section();
mtx.unlock();
```

### 2. 임계 영역 최소화
```cpp
// 나쁨: 긴 임계 영역
{
    std::lock_guard<std::mutex> lock(mtx);
    expensive_computation();  // 이 동안 잠금을 유지하지 마세요!
    shared_data = result;
}

// 좋음: 최소한의 임계 영역
auto result = expensive_computation();
{
    std::lock_guard<std::mutex> lock(mtx);
    shared_data = result;
}
```

### 3. 데드락 방지를 위한 잠금 순서
```cpp
// 좋음: 항상 같은 순서로 잠금
void transfer(Account& from, Account& to, double amount) {
    // 낮은 주소 먼저 잠금
    std::mutex* first = &from.mtx < &to.mtx ? &from.mtx : &to.mtx;
    std::mutex* second = &from.mtx < &to.mtx ? &to.mtx : &from.mtx;

    std::lock_guard<std::mutex> lock1(*first);
    std::lock_guard<std::mutex> lock2(*second);

    from.balance -= amount;
    to.balance += amount;
}

// 더 좋음: scoped_lock 사용
void transfer(Account& from, Account& to, double amount) {
    std::scoped_lock lock(from.mtx, to.mtx);
    from.balance -= amount;
    to.balance += amount;
}
```

### 4. 읽기 위주 워크로드에는 Shared Mutex 사용
```cpp
class Cache {
    mutable std::shared_mutex mtx;
    std::map<std::string, std::string> data;

public:
    // 다수의 reader - shared lock 사용
    std::string get(const std::string& key) const {
        std::shared_lock lock(mtx);
        return data.at(key);
    }

    // 소수의 writer - unique lock 사용
    void set(const std::string& key, const std::string& value) {
        std::unique_lock lock(mtx);
        data[key] = value;
    }
};
```

### 5. 가능하면 재귀 Mutex 피하기
```cpp
// 나쁨: 설계 문제를 가리기 위해 재귀 mutex 사용
class BadDesign {
    std::recursive_mutex mtx;

    void foo() {
        std::lock_guard lock(mtx);
        bar();  // 다시 잠금
    }

    void bar() {
        std::lock_guard lock(mtx);
        // 작업
    }
};

// 좋음: 재귀를 피하도록 리팩터링
class GoodDesign {
    std::mutex mtx;

    void bar_internal() {
        // 작업 (잠금이 유지되고 있다고 가정)
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

## 일반적인 실수

### 1. 잠금 잊기
```cpp
// 나쁨: 동기화 없음
class UnsafeCounter {
    int value = 0;
public:
    void increment() { ++value; }  // 경쟁 조건!
};

// 좋음
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

### 2. 여러 잠금으로 인한 데드락
```cpp
// 나쁨: 데드락 발생 가능
std::mutex m1, m2;

void thread1() {
    std::lock_guard lock1(m1);
    std::lock_guard lock2(m2);
}

void thread2() {
    std::lock_guard lock2(m2);  // 순서가 반대!
    std::lock_guard lock1(m1);
}

// 좋음: scoped_lock 사용
void thread1() {
    std::scoped_lock lock(m1, m2);
}

void thread2() {
    std::scoped_lock lock(m1, m2);  // 순서는 중요하지 않음
}
```

### 3. 너무 많이 잠그기
```cpp
// 나쁨: I/O 중에 잠금 유지
{
    std::lock_guard lock(mtx);
    std::cout << shared_data << "\n";  // 잠금 상태로 I/O!
}

// 좋음: 데이터 복사, 잠금 해제 후 I/O
std::string data_copy;
{
    std::lock_guard lock(mtx);
    data_copy = shared_data;
}
std::cout << data_copy << "\n";
```

### 4. 모든 접근을 보호하지 않음
```cpp
// 나쁨: 일관되지 않은 보호
class BadCache {
    std::mutex mtx;
    std::map<int, int> data;

public:
    void set(int key, int value) {
        std::lock_guard lock(mtx);
        data[key] = value;
    }

    int get(int key) {
        return data[key];  // 잠금을 잊었음!
    }
};
```

### 5. 보호된 데이터에 대한 참조 반환
```cpp
// 나쁨: 보호된 데이터를 노출
class BadContainer {
    std::mutex mtx;
    std::vector<int> data;

public:
    std::vector<int>& get_data() {
        std::lock_guard lock(mtx);
        return data;  // 잠금이 해제되었지만 참조가 탈출!
    }
};

// 좋음: 복사본 반환
class GoodContainer {
    std::mutex mtx;
    std::vector<int> data;

public:
    std::vector<int> get_data() {
        std::lock_guard lock(mtx);
        return data;  // 복사
    }
};
```

## 내부 메커니즘

### std::mutex Futex 구현 (Linux/glibc)

```cpp
// pthread_mutex 내부 구조 (glibc NPTL)
struct __pthread_mutex_s {
    int __lock;           // 0: unlocked, 1: locked, 2: contended
    unsigned int __count; // recursive lock count
    int __owner;          // owning thread ID (for recursive/errorcheck)
    // ... 견고성, 우선순위 등의 추가 필드
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

### scoped_lock 데드락 회피 알고리즘

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

### shared_mutex Reader-Writer 구현

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

### Recursive Mutex 카운터 오버플로우

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

## 성능 고려사항

### 잠금 오버헤드
- **비경합 잠금**: ~25 나노초
- **경합 잠금**: 1000배 더 느릴 수 있음 (마이크로초)
- **컨텍스트 스위치**: 1-10 마이크로초

### 잠금 세분화
```cpp
// 세분화됨: 더 많은 병렬성, 더 많은 오버헤드
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

// 조분화됨: 더 적은 오버헤드, 더 적은 병렬성
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

## 전체 예제: 스레드 안전 큐
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

    // 생산자
    std::thread producer([&queue] {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
    });

    // 소비자
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

## 추가 읽기

- [C++ Reference: std::mutex](https://en.cppreference.com/w/cpp/thread/mutex)
- [C++ Reference: std::lock_guard](https://en.cppreference.com/w/cpp/thread/lock_guard)
- [Condition Variable](./04-condition-variable.md)

## 탐색

- [C++ 개요로 돌아가기](./README.md)
- 이전: [std::thread](./01-std-thread.md)
- 다음: [Atomic 연산](./03-atomic.md)
