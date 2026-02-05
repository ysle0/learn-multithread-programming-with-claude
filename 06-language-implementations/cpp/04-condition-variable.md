# C++의 Condition Variable

Condition variable은 스레드가 특정 조건이 참이 될 때까지 대기할 수 있게 하여, 바쁜 대기(busy-waiting) 없이 스레드 간의 효율적인 통신과 동기화를 가능하게 합니다.

## 목차
- [기본 개념](#기본-개념)
- [std::condition_variable](#stdcondition_variable)
- [대기 연산](#대기-연산)
- [알림 연산](#알림-연산)
- [일반적인 패턴](#일반적인-패턴)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### Condition Variable이란?

Condition variable은 스레드가 다음을 할 수 있게 합니다:
1. 조건이 참이 될 때까지 **대기** (스레드 블로킹)
2. 조건이 변경되었을 때 **신호** 보내기 (대기 중인 스레드 깨우기)

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
    cv.wait(lock, [] { return ready; });  // ready가 true가 될 때까지 대기
    std::cout << "Proceeding!\n";
}

void send_signal() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one();  // 대기 중인 스레드 하나를 깨움
}

int main() {
    std::thread t1(wait_for_signal);
    std::thread t2(send_signal);
    t1.join();
    t2.join();
    return 0;
}
```

### Condition Variable을 사용하는 이유

**CV 없이 (바쁜 대기 - 나쁨):**
```cpp
// 나쁨: CPU를 낭비
while (!ready) {
    std::this_thread::yield();  // 여전히 바쁜 대기
}
```

**CV 사용 (효율적 - 좋음):**
```cpp
// 좋음: 알림을 받을 때까지 스레드가 sleep
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

## std::condition_variable

### 기본 구조

```cpp
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool condition = false;

void waiter() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return condition; });
    // 조건이 이제 true이고, 잠금이 유지됨
}

void notifier() {
    {
        std::lock_guard<std::mutex> lock(mtx);
        condition = true;
    }  // notify 전에 잠금 해제
    cv.notify_one();
}
```

### 요구사항

1. `std::unique_lock`을 사용해야 함 (`std::lock_guard` 아님)
2. 조건을 mutex로 보호해야 함
3. 가짜 깨어남을 피하기 위해 서술어를 사용해야 함

## 대기 연산

### 서술어가 있는 wait()

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

    // ready가 true가 될 때까지 대기
    cv.wait(lock, [] { return ready; });

    // 이제 data를 안전하게 사용할 수 있음
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

### 서술어 없는 wait() (수동 루프)

```cpp
#include <mutex>
#include <condition_variable>

std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void wait_manual() {
    std::unique_lock<std::mutex> lock(mtx);

    // 가짜 깨어남을 처리하기 위해 루프 필요
    while (!ready) {
        cv.wait(lock);  // 잠금을 해제하고 sleep
    }                   // 깨어날 때 잠금을 다시 획득

    // ready가 이제 true
}
```

### wait_for() - 시간 제한 대기

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
    // 신호를 보내지 않고, 타임아웃되게 함
    t.join();
    return 0;
}
```

### wait_until() - 특정 시점까지 대기

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

## 알림 연산

### notify_one() - 스레드 하나 깨우기

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

    cv.notify_one();  // 스레드 하나만 깨어남
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    cv.notify_one();  // 두 번째 스레드 깨우기

    t1.join();
    t2.join();
    return 0;
}
```

### notify_all() - 모든 스레드 깨우기

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

    cv.notify_all();  // 모든 대기 중인 스레드 깨우기

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### notify_one vs. notify_all 사용 시점

```cpp
// notify_one 사용 시점:
// - 하나의 스레드만 이벤트를 처리해야 할 때
// - 예: 여러 워커가 있는 작업 큐

cv.notify_one();  // 워커 하나만 깨우기

// notify_all 사용 시점:
// - 모든 스레드가 이벤트에 반응해야 할 때
// - 예: 배리어 동기화

cv.notify_all();  // 모든 스레드 깨우기
```

## 일반적인 패턴

### 생산자-소비자 큐

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
            return false;  // 큐 종료
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

    // 생산자
    std::thread producer([&queue] {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
        queue.finish();
    });

    // 소비자
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

### 배리어 동기화

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
    barrier.wait();  // 동기화

    std::cout << "Thread " << id << " phase 2\n";
    barrier.wait();  // 다시 동기화

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

### Condition Variable을 사용한 스레드 풀

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

### 이벤트 시그널링

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

## 다른 언어와의 비교

### C++ vs. C#
```cpp
// C++
std::mutex mtx;
std::condition_variable cv;
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });

// C# 동등 코드:
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

// Go - 채널 (다른 패러다임)
// <-readyChan
```

### C++ vs. JavaScript
```cpp
// C++는 condition variable을 가짐
std::condition_variable cv;

// JavaScript에는 condition variable이 없음
// (단일 스레드 메인 실행)
// 비동기 조정에는 Promise 사용
```

## 모범 사례

### 1. 항상 서술어 사용

```cpp
// 나쁨: 가짜 깨어남을 처리하지 않음
cv.wait(lock);
// 조건이 true가 아닌데 깨어날 수 있음!

// 좋음: 서술어가 가짜 깨어남을 처리
cv.wait(lock, [] { return ready; });
```

### 2. Mutex로 조건 보호

```cpp
// 나쁨: 조건이 보호되지 않음
ready = true;
cv.notify_one();  // 경쟁 조건!

// 좋음: mutex로 조건 보호
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}
cv.notify_one();
```

### 3. 가능하면 잠금 바깥에서 알림

```cpp
// 좋음: 잠금 해제 후 알림
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
}  // 잠금 해제
cv.notify_one();  // 그 다음 알림

// 괜찮음: 잠금 안에서 알림 (하지만 덜 효율적)
{
    std::lock_guard<std::mutex> lock(mtx);
    ready = true;
    cv.notify_one();
}
```

### 4. lock_guard가 아닌 unique_lock 사용

```cpp
// 필수: cv.wait는 unique_lock이 필요
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });

// 컴파일 안 됨: lock_guard는 unlock/lock을 지원하지 않음
// std::lock_guard<std::mutex> lock(mtx);
// cv.wait(lock, [] { return ready; });  // 에러
```

### 5. 바쁜 대기 피하기

```cpp
// 나쁨: 바쁜 대기
while (!ready) {
    std::this_thread::sleep_for(std::chrono::milliseconds(10));
}

// 좋음: condition variable 사용
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

## 일반적인 실수

### 1. 가짜 깨어남

```cpp
// 나쁨: 깨어남이 곧 조건이 true라고 가정
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock);  // 가짜로 깨어날 수 있음!
    // ready가 여전히 false일 수 있음
}

// 좋음: 루프/서술어로 조건 확인
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
    // ready가 true임이 보장됨
}
```

### 2. 놓친 깨어남

```cpp
// 나쁨: 대기 전에 알림
void thread1() {
    ready = true;
    cv.notify_one();  // 아무도 대기하지 않는데 보냄!
}

void thread2() {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });  // 신호를 놓쳤을 수 있음
}

// 좋음: 적절한 동기화 사용
// 서술어(ready)가 알림이 먼저 발생해도 정확성을 보장
```

### 3. notify_one으로 인한 데드락

```cpp
// 미묘함: 모든 스레드가 다른 조건을 대기할 수 있음
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
    cv.notify_one();  // thread2를 깨울 수 있고, thread2는 다시 sleep!
}

// 해결책: notify_all 사용 또는 별도의 condition variable 사용
```

### 4. 조건 확인 시 잠금 미보유

```cpp
// 나쁨: 경쟁 조건
if (ready) {  // 잠금 없이 확인!
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
}

// 좋음: 잠금을 유지한 채 조건 확인
{
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; });
}
```

### 5. 예외 안전성

```cpp
// 나쁨: 예외 발생 시 잠금이 해제되지 않음
std::unique_lock<std::mutex> lock(mtx);
might_throw();  // 예외 발생 시 잠금 유지!
cv.wait(lock, [] { return ready; });

// 좋음: RAII가 잠금 해제를 보장
{
    std::unique_lock<std::mutex> lock(mtx);
    try {
        might_throw();
    } catch (...) {
        // 잠금이 자동으로 해제됨
        throw;
    }
    cv.wait(lock, [] { return ready; });
}

// 더 좋음: RAII에 의존
{
    std::unique_lock<std::mutex> lock(mtx);
    might_throw();  // 예외 시 잠금 해제됨
    cv.wait(lock, [] { return ready; });
}
```

## 내부 메커니즘

### Futex 기반 구현 (Linux)

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

## 성능 고려사항

### 연산 비용

```cpp
// cv.wait() - 비교적 비용이 큼
// - Mutex unlock
// - 스레드 컨텍스트 스위치 (sleep으로)
// - 스레드 컨텍스트 스위치 (깨어날 때)
// - Mutex lock

// 현명하게 사용:
// - 좋음: 빈도가 낮은 이벤트에 적합
// - 나쁨: 빈도가 높은 시그널링 (대신 atomic 고려)
```

### Thundering Herd 문제

```cpp
// 문제: notify_all이 많은 스레드를 깨우지만 하나만 진행 가능
std::condition_variable cv;
bool work_available = false;

// 다수의 워커:
cv.wait(lock, [] { return work_available; });
// 모두 깨어나지만, 하나만 작업을 가져감

// 해결책: 작업 큐에 notify_one 사용
cv.notify_one();  // 워커 하나만 깨우기
```

## 전체 예제: 세마포어 구현

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

Semaphore sem(2);  // 최대 2개의 동시 접근

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

## 추가 읽기

- [C++ Reference: std::condition_variable](https://en.cppreference.com/w/cpp/thread/condition_variable)
- [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
- [Async와 Future](./05-async-future.md)

## 탐색

- [C++ 개요로 돌아가기](./README.md)
- 이전: [Atomic 연산](./03-atomic.md)
- 다음: [Async와 Future](./05-async-future.md)
