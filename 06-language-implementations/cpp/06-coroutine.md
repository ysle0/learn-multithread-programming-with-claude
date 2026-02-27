# C++20 Coroutine (코루틴)

## 개요

**Coroutine**은 C++20에서 도입된 일급 시민으로, 실행을 일시 중단(suspend)하고 나중에 재개(resume)할 수 있는 함수입니다. 전통적인 함수와 달리 coroutine은 여러 진입점과 탈출점을 가지며, 비동기 프로그래밍, 지연 평가, 제너레이터 패턴 등에 활용됩니다.

### 일반 함수 vs Coroutine

```
일반 함수:
┌─────────────┐
│   호출      │───────────────────────────▶│ 완료/반환 │
│   (call)    │    연속 실행 (blocking)     │  (return) │
└─────────────┘                             └───────────┘

Coroutine:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   호출      │───▶│  일시 중단   │───▶│   재개      │───▶ ...
│   (call)    │    │  (suspend)  │    │  (resume)   │
└─────────────┘    └─────────────┘    └─────────────┘
     │                   │                   │
     ▼                   ▼                   ▼
   실행               상태 보존            실행 계속
```

## 핵심 키워드

C++ coroutine은 세 가지 키워드로 정의됩니다:

| 키워드 | 설명 |
|--------|------|
| `co_await` | 비동기 작업 완료를 기다리며 일시 중단 |
| `co_yield` | 값을 반환하고 일시 중단 (제너레이터 패턴) |
| `co_return` | coroutine 종료 및 최종 값 반환 |

함수 본문에 이 키워드 중 하나라도 있으면 해당 함수는 coroutine이 됩니다.

## 기본 구성 요소

### 1. Promise Type

Coroutine의 동작을 제어하는 타입입니다.

```cpp
struct MyPromise {
    // Coroutine 반환 객체 생성
    MyCoroutine get_return_object();

    // 시작 시 동작: 즉시 실행 또는 일시 중단
    std::suspend_always initial_suspend();  // 시작 시 중단
    // std::suspend_never initial_suspend(); // 즉시 실행

    // 종료 시 동작
    std::suspend_always final_suspend() noexcept;

    // co_return 처리
    void return_void();                     // co_return;
    // void return_value(T value);          // co_return value;

    // co_yield 처리
    std::suspend_always yield_value(T value);

    // 예외 처리
    void unhandled_exception();
};
```

### 2. Coroutine Handle

Coroutine 인스턴스를 제어하는 핸들입니다.

```cpp
#include <coroutine>

// 특정 promise 타입에 대한 핸들
std::coroutine_handle<MyPromise> handle;

// 타입 소거된 핸들
std::coroutine_handle<> generic_handle;

// 주요 메서드
handle.resume();      // 실행 재개
handle.destroy();     // coroutine 파괴
handle.done();        // 완료 여부 확인
handle.promise();     // promise 객체 접근
```

### 3. Awaitable/Awaiter

`co_await` 표현식에서 사용되는 객체입니다.

```cpp
struct MyAwaiter {
    // 즉시 준비되었는지 확인 (true면 suspend 안 함)
    bool await_ready() const noexcept { return false; }

    // 일시 중단 시 호출 (true 반환 시 중단, false면 즉시 재개)
    void await_suspend(std::coroutine_handle<> h) {
        // 비동기 작업 시작, 완료 시 h.resume() 호출
    }

    // 재개 시 호출, co_await의 결과값 반환
    T await_resume() { return result; }
};
```

## 간단한 제너레이터 구현

### Generator 클래스

```cpp
#include <coroutine>
#include <iostream>
#include <optional>

template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;

        Generator get_return_object() {
            return Generator{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }

        // 시작 시 일시 중단 (lazy evaluation)
        std::suspend_always initial_suspend() { return {}; }

        // 종료 시 일시 중단 (handle.done() 확인 가능)
        std::suspend_always final_suspend() noexcept { return {}; }

        // co_yield value; 처리
        std::suspend_always yield_value(T value) {
            current_value = std::move(value);
            return {};
        }

        void return_void() {}

        void unhandled_exception() {
            std::terminate();
        }
    };

    using Handle = std::coroutine_handle<promise_type>;

    explicit Generator(Handle h) : handle_(h) {}

    ~Generator() {
        if (handle_) handle_.destroy();
    }

    // 이동만 허용
    Generator(Generator&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }
    Generator& operator=(Generator&& other) noexcept {
        if (this != &other) {
            if (handle_) handle_.destroy();
            handle_ = other.handle_;
            other.handle_ = nullptr;
        }
        return *this;
    }

    // 복사 금지
    Generator(const Generator&) = delete;
    Generator& operator=(const Generator&) = delete;

    // Iterator 인터페이스
    class Iterator {
    public:
        using iterator_category = std::input_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;

        Iterator() : handle_(nullptr) {}
        explicit Iterator(Handle h) : handle_(h) {}

        Iterator& operator++() {
            handle_.resume();
            if (handle_.done()) handle_ = nullptr;
            return *this;
        }

        T& operator*() { return handle_.promise().current_value; }

        bool operator==(const Iterator& other) const {
            return handle_ == other.handle_;
        }
        bool operator!=(const Iterator& other) const {
            return !(*this == other);
        }

    private:
        Handle handle_;
    };

    Iterator begin() {
        if (handle_) {
            handle_.resume();
            if (handle_.done()) return end();
        }
        return Iterator{handle_};
    }

    Iterator end() { return Iterator{}; }

    // 다음 값 가져오기
    std::optional<T> next() {
        if (!handle_ || handle_.done()) return std::nullopt;
        handle_.resume();
        if (handle_.done()) return std::nullopt;
        return handle_.promise().current_value;
    }

private:
    Handle handle_;
};
```

### 사용 예제

```cpp
// 피보나치 수열 제너레이터
Generator<int> fibonacci(int limit) {
    int a = 0, b = 1;
    while (a < limit) {
        co_yield a;
        int next = a + b;
        a = b;
        b = next;
    }
}

// 범위 제너레이터
Generator<int> range(int start, int end) {
    for (int i = start; i < end; ++i) {
        co_yield i;
    }
}

int main() {
    std::cout << "피보나치 수열 (< 100):\n";
    for (int value : fibonacci(100)) {
        std::cout << value << " ";
    }
    std::cout << "\n";

    std::cout << "\n범위 [0, 5):\n";
    auto gen = range(0, 5);
    while (auto value = gen.next()) {
        std::cout << *value << " ";
    }
    std::cout << "\n";

    return 0;
}

// 출력:
// 피보나치 수열 (< 100):
// 0 1 1 2 3 5 8 13 21 34 55 89
//
// 범위 [0, 5):
// 0 1 2 3 4
```

## 비동기 Task 구현

### Task 클래스

```cpp
#include <coroutine>
#include <exception>
#include <variant>

template<typename T>
class Task {
public:
    struct promise_type {
        std::variant<std::monostate, T, std::exception_ptr> result;
        std::coroutine_handle<> continuation;

        Task get_return_object() {
            return Task{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }

        // 즉시 실행 시작
        std::suspend_never initial_suspend() { return {}; }

        // 종료 시 continuation 재개
        auto final_suspend() noexcept {
            struct FinalAwaiter {
                bool await_ready() noexcept { return false; }

                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    auto& promise = h.promise();
                    if (promise.continuation) {
                        return promise.continuation;
                    }
                    return std::noop_coroutine();
                }

                void await_resume() noexcept {}
            };
            return FinalAwaiter{};
        }

        void return_value(T value) {
            result.template emplace<1>(std::move(value));
        }

        void unhandled_exception() {
            result.template emplace<2>(std::current_exception());
        }
    };

    using Handle = std::coroutine_handle<promise_type>;

    explicit Task(Handle h) : handle_(h) {}

    ~Task() {
        if (handle_) handle_.destroy();
    }

    Task(Task&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }

    // co_await 지원
    auto operator co_await() {
        struct TaskAwaiter {
            Handle handle;

            bool await_ready() {
                return handle.done();
            }

            std::coroutine_handle<> await_suspend(
                std::coroutine_handle<> continuation) {
                handle.promise().continuation = continuation;
                return handle;
            }

            T await_resume() {
                auto& result = handle.promise().result;
                if (std::holds_alternative<std::exception_ptr>(result)) {
                    std::rethrow_exception(std::get<std::exception_ptr>(result));
                }
                return std::move(std::get<T>(result));
            }
        };
        return TaskAwaiter{handle_};
    }

    // 결과 가져오기 (blocking)
    T get() {
        // 완료될 때까지 대기
        while (!handle_.done()) {
            // 실제 구현에서는 이벤트 루프 사용
        }

        auto& result = handle_.promise().result;
        if (std::holds_alternative<std::exception_ptr>(result)) {
            std::rethrow_exception(std::get<std::exception_ptr>(result));
        }
        return std::move(std::get<T>(result));
    }

private:
    Handle handle_;
};

// void 특수화
template<>
class Task<void> {
public:
    struct promise_type {
        std::exception_ptr exception;
        std::coroutine_handle<> continuation;

        Task get_return_object() {
            return Task{
                std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }

        std::suspend_never initial_suspend() { return {}; }

        auto final_suspend() noexcept {
            struct FinalAwaiter {
                bool await_ready() noexcept { return false; }
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                    if (h.promise().continuation)
                        return h.promise().continuation;
                    return std::noop_coroutine();
                }
                void await_resume() noexcept {}
            };
            return FinalAwaiter{};
        }

        void return_void() {}

        void unhandled_exception() {
            exception = std::current_exception();
        }
    };

    using Handle = std::coroutine_handle<promise_type>;

    explicit Task(Handle h) : handle_(h) {}
    ~Task() { if (handle_) handle_.destroy(); }
    Task(Task&& other) noexcept : handle_(other.handle_) {
        other.handle_ = nullptr;
    }

    auto operator co_await() {
        struct TaskAwaiter {
            Handle handle;
            bool await_ready() { return handle.done(); }
            std::coroutine_handle<> await_suspend(
                std::coroutine_handle<> continuation) {
                handle.promise().continuation = continuation;
                return handle;
            }
            void await_resume() {
                if (handle.promise().exception)
                    std::rethrow_exception(handle.promise().exception);
            }
        };
        return TaskAwaiter{handle_};
    }

private:
    Handle handle_;
};
```

### 비동기 작업 예제

```cpp
// 비동기 덧셈
Task<int> async_add(int a, int b) {
    co_return a + b;
}

// 비동기 연쇄 호출
Task<int> compute() {
    int x = co_await async_add(10, 20);
    int y = co_await async_add(x, 30);
    co_return y;
}

// 메인 coroutine
Task<void> main_task() {
    int result = co_await compute();
    std::cout << "결과: " << result << "\n";  // 출력: 결과: 60
}
```

## Awaiter 상세 분석

### co_await 표현식의 동작

```cpp
// co_await expr; 의 변환 과정

// 1. Awaitable 객체 획득
auto&& awaitable = expr;

// 2. Awaiter 객체 획득
auto&& awaiter = get_awaiter(awaitable);

// 3. await_ready() 확인
if (!awaiter.await_ready()) {
    // 4. 일시 중단
    <suspend coroutine>

    // 5. await_suspend() 호출
    auto result = awaiter.await_suspend(handle);

    // result 타입에 따른 동작:
    // - void: 제어권 반환
    // - bool: true면 중단, false면 즉시 재개
    // - coroutine_handle<>: 해당 핸들 재개 (symmetric transfer)
}

// 6. 재개 후 await_resume() 호출
return awaiter.await_resume();
```

### 사용자 정의 Awaiter 예제

```cpp
#include <chrono>
#include <thread>

// 타이머 awaiter
class SleepAwaiter {
public:
    explicit SleepAwaiter(std::chrono::milliseconds duration)
        : duration_(duration) {}

    bool await_ready() const noexcept {
        // 항상 일시 중단
        return false;
    }

    void await_suspend(std::coroutine_handle<> handle) {
        // 별도 스레드에서 타이머 실행
        std::thread([this, handle]() {
            std::this_thread::sleep_for(duration_);
            handle.resume();  // 타이머 완료 후 재개
        }).detach();
    }

    void await_resume() const noexcept {
        // 반환값 없음
    }

private:
    std::chrono::milliseconds duration_;
};

// 편의 함수
SleepAwaiter sleep_for(std::chrono::milliseconds duration) {
    return SleepAwaiter{duration};
}

// 사용 예
Task<void> timer_example() {
    std::cout << "시작\n";
    co_await sleep_for(std::chrono::milliseconds(1000));
    std::cout << "1초 후\n";
    co_await sleep_for(std::chrono::milliseconds(500));
    std::cout << "0.5초 더 후\n";
}
```

### 이벤트 기반 Awaiter

```cpp
#include <functional>
#include <queue>
#include <mutex>

// 간단한 이벤트 큐
class EventQueue {
public:
    void push(std::coroutine_handle<> handle) {
        std::lock_guard lock(mutex_);
        queue_.push(handle);
    }

    void process_one() {
        std::coroutine_handle<> handle;
        {
            std::lock_guard lock(mutex_);
            if (queue_.empty()) return;
            handle = queue_.front();
            queue_.pop();
        }
        handle.resume();
    }

    void process_all() {
        while (!empty()) {
            process_one();
        }
    }

    bool empty() {
        std::lock_guard lock(mutex_);
        return queue_.empty();
    }

private:
    std::queue<std::coroutine_handle<>> queue_;
    std::mutex mutex_;
};

// 전역 이벤트 큐
EventQueue g_event_queue;

// 스케줄 awaiter
struct ScheduleAwaiter {
    bool await_ready() const noexcept { return false; }

    void await_suspend(std::coroutine_handle<> handle) {
        g_event_queue.push(handle);
    }

    void await_resume() const noexcept {}
};

// 다음 이벤트 루프 사이클로 양보
ScheduleAwaiter yield() {
    return {};
}

// 사용 예
Task<void> cooperative_task(int id) {
    for (int i = 0; i < 3; ++i) {
        std::cout << "Task " << id << ": 반복 " << i << "\n";
        co_await yield();  // 다른 태스크에게 양보
    }
}

void run_scheduler() {
    cooperative_task(1);
    cooperative_task(2);

    // 이벤트 루프
    while (!g_event_queue.empty()) {
        g_event_queue.process_one();
    }
}

// 출력 (인터리브됨):
// Task 1: 반복 0
// Task 2: 반복 0
// Task 1: 반복 1
// Task 2: 반복 1
// Task 1: 반복 2
// Task 2: 반복 2
```

## Symmetric Transfer

### 개념

Symmetric transfer는 한 coroutine에서 다른 coroutine으로 직접 제어를 전달하는 최적화 기법입니다. 스택 오버플로우를 방지하고 성능을 향상시킵니다.

```
일반적인 재개 (스택 증가):
┌────────────┐
│ Coroutine A│ ─────────────────────────────────┐
└────────────┘                                  │
                                                ▼
                                         ┌────────────┐
                                         │ Coroutine B│ ────────────┐
                                         └────────────┘             │
                                                                    ▼
                                                             ┌────────────┐
                                                             │ Coroutine C│
                                                             └────────────┘
스택: A → B → C (스택 깊이 증가)

Symmetric Transfer (스택 일정):
┌────────────┐     ┌────────────┐     ┌────────────┐
│ Coroutine A│ ──▶ │ Coroutine B│ ──▶ │ Coroutine C│
└────────────┘     └────────────┘     └────────────┘

스택: 항상 깊이 1
```

### 구현

```cpp
struct SymmetricAwaiter {
    std::coroutine_handle<> target;

    bool await_ready() const noexcept { return false; }

    // coroutine_handle 반환 → symmetric transfer
    std::coroutine_handle<> await_suspend(
        std::coroutine_handle<>) noexcept {
        return target;
    }

    void await_resume() const noexcept {}
};

// 다른 coroutine으로 직접 전환
SymmetricAwaiter switch_to(std::coroutine_handle<> target) {
    return SymmetricAwaiter{target};
}
```

## Coroutine과 멀티스레딩

### Thread-safe 스케줄러

```cpp
#include <thread>
#include <vector>
#include <atomic>
#include <condition_variable>

class ThreadPoolScheduler {
public:
    explicit ThreadPoolScheduler(size_t num_threads)
        : running_(true) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }

    ~ThreadPoolScheduler() {
        {
            std::lock_guard lock(mutex_);
            running_ = false;
        }
        cv_.notify_all();
        for (auto& worker : workers_) {
            worker.join();
        }
    }

    void schedule(std::coroutine_handle<> handle) {
        {
            std::lock_guard lock(mutex_);
            queue_.push(handle);
        }
        cv_.notify_one();
    }

    auto schedule_awaiter() {
        struct Awaiter {
            ThreadPoolScheduler* scheduler;

            bool await_ready() const noexcept { return false; }

            void await_suspend(std::coroutine_handle<> handle) {
                scheduler->schedule(handle);
            }

            void await_resume() const noexcept {}
        };
        return Awaiter{this};
    }

private:
    void worker_loop() {
        while (true) {
            std::coroutine_handle<> handle;
            {
                std::unique_lock lock(mutex_);
                cv_.wait(lock, [this] {
                    return !running_ || !queue_.empty();
                });

                if (!running_ && queue_.empty()) return;

                handle = queue_.front();
                queue_.pop();
            }
            handle.resume();
        }
    }

    std::vector<std::thread> workers_;
    std::queue<std::coroutine_handle<>> queue_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool running_;
};

// 사용 예
ThreadPoolScheduler scheduler(4);

Task<int> parallel_work(int id) {
    co_await scheduler.schedule_awaiter();

    // 이제 worker 스레드에서 실행
    std::cout << "Task " << id << " on thread "
              << std::this_thread::get_id() << "\n";

    // CPU 작업
    int result = 0;
    for (int i = 0; i < 1000000; ++i) {
        result += i;
    }

    co_return result;
}
```

### Mutex와 Coroutine

```cpp
// Coroutine-aware mutex
class AsyncMutex {
public:
    class LockGuard {
    public:
        explicit LockGuard(AsyncMutex& mutex) : mutex_(mutex) {}
        ~LockGuard() { mutex_.unlock(); }

        LockGuard(const LockGuard&) = delete;
        LockGuard& operator=(const LockGuard&) = delete;

    private:
        AsyncMutex& mutex_;
    };

    auto lock() {
        struct LockAwaiter {
            AsyncMutex* mutex;

            bool await_ready() {
                // lock 시도
                bool expected = false;
                return mutex->locked_.compare_exchange_strong(
                    expected, true, std::memory_order_acquire);
            }

            void await_suspend(std::coroutine_handle<> handle) {
                // 대기 큐에 추가
                std::lock_guard lock(mutex->queue_mutex_);
                mutex->waiters_.push(handle);
            }

            LockGuard await_resume() {
                return LockGuard{*mutex};
            }
        };
        return LockAwaiter{this};
    }

private:
    void unlock() {
        std::coroutine_handle<> to_resume;
        {
            std::lock_guard lock(queue_mutex_);
            if (waiters_.empty()) {
                locked_.store(false, std::memory_order_release);
                return;
            }
            to_resume = waiters_.front();
            waiters_.pop();
        }
        to_resume.resume();
    }

    std::atomic<bool> locked_{false};
    std::queue<std::coroutine_handle<>> waiters_;
    std::mutex queue_mutex_;
};

// 사용 예
AsyncMutex mutex;

Task<void> critical_section(int id) {
    auto guard = co_await mutex.lock();
    std::cout << "Task " << id << " in critical section\n";
    co_await sleep_for(std::chrono::milliseconds(100));
    std::cout << "Task " << id << " leaving critical section\n";
}
```

## 내부 메커니즘

### Coroutine Frame 구조

컴파일러가 coroutine을 변환할 때 생성하는 프레임 구조입니다.

```cpp
// 원본 coroutine
Generator<int> example_coro(int start) {
    int x = start;
    co_yield x++;
    co_yield x++;
    co_yield x;
}

// 컴파일러가 생성하는 프레임 (개념적)
struct __example_coro_frame {
    // Resume/Destroy 함수 포인터
    void (*__resume_fn)(__example_coro_frame*);
    void (*__destroy_fn)(__example_coro_frame*);

    // Promise 객체
    Generator<int>::promise_type __promise;

    // 로컬 변수
    int start;
    int x;

    // 재개 지점 (suspension point index)
    int __suspend_index;

    // 초기 awaiter 저장
    std::suspend_always __initial_awaiter;

    // yield awaiter 저장
    std::suspend_always __yield_awaiter;

    // 최종 awaiter 저장
    std::suspend_always __final_awaiter;
};

// 상태 머신으로 변환된 resume 함수
void __example_coro_resume(__example_coro_frame* __frame) {
    switch (__frame->__suspend_index) {
    case 0:  // initial_suspend 후
        __frame->x = __frame->start;

        // 첫 번째 co_yield
        __frame->__promise.yield_value(__frame->x++);
        __frame->__suspend_index = 1;
        return;

    case 1:  // 첫 번째 yield 후
        // 두 번째 co_yield
        __frame->__promise.yield_value(__frame->x++);
        __frame->__suspend_index = 2;
        return;

    case 2:  // 두 번째 yield 후
        // 세 번째 co_yield
        __frame->__promise.yield_value(__frame->x);
        __frame->__suspend_index = 3;
        return;

    case 3:  // 세 번째 yield 후
        // final_suspend
        __frame->__suspend_index = 4;
        return;
    }
}
```

### 메모리 레이아웃

```
Coroutine Frame:
┌─────────────────────────────────────────────────────┐
│  Resume Function Pointer (8 bytes)                  │
├─────────────────────────────────────────────────────┤
│  Destroy Function Pointer (8 bytes)                 │
├─────────────────────────────────────────────────────┤
│  Promise Object                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │  current_value: T                              │ │
│  │  exception_ptr: std::exception_ptr            │ │
│  │  ...                                          │ │
│  └───────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│  Suspend Index (4 bytes)                            │
├─────────────────────────────────────────────────────┤
│  Local Variables                                    │
│  ┌───────────────────────────────────────────────┐ │
│  │  start: int (4 bytes)                          │ │
│  │  x: int (4 bytes)                              │ │
│  │  temp_awaiter: ... (awaiter 임시 저장)         │ │
│  └───────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────┤
│  Padding (alignment)                                │
└─────────────────────────────────────────────────────┘
```

### HALO (Heap Allocation eLision Optimization)

컴파일러가 coroutine frame 힙 할당을 제거하는 최적화입니다.

```cpp
// HALO 적용 조건:
// 1. Coroutine의 수명이 호출자 내에서 완전히 결정됨
// 2. Coroutine 크기가 컴파일 타임에 결정됨
// 3. Coroutine이 인라인되거나 수명 추적 가능

// HALO 적용 가능한 패턴
Task<int> caller() {
    // callee의 프레임이 caller 스택에 할당될 수 있음
    int result = co_await callee();
    co_return result;
}

// HALO 적용 불가능한 패턴
void store_coro(std::vector<Task<int>>& tasks) {
    // coroutine이 함수 범위를 벗어남
    tasks.push_back(async_work());  // 힙 할당 필요
}
```

### Coroutine Handle 내부

```cpp
// std::coroutine_handle 구현 (개념적)
template<typename Promise = void>
class coroutine_handle {
public:
    // 프레임 포인터
    void* __frame_ptr_ = nullptr;

    // Promise로부터 handle 생성
    static coroutine_handle from_promise(Promise& promise) {
        coroutine_handle h;
        // Promise는 프레임 내에 고정 오프셋에 위치
        h.__frame_ptr_ = reinterpret_cast<char*>(&promise)
                        - __promise_offset();
        return h;
    }

    // 재개
    void resume() const {
        auto* frame = static_cast<__coroutine_frame_base*>(__frame_ptr_);
        frame->__resume_fn(frame);
    }

    // 파괴
    void destroy() const {
        auto* frame = static_cast<__coroutine_frame_base*>(__frame_ptr_);
        frame->__destroy_fn(frame);
    }

    // 완료 확인
    bool done() const {
        auto* frame = static_cast<__coroutine_frame_base*>(__frame_ptr_);
        return frame->__suspend_index == __final_index;
    }

    // Promise 접근
    Promise& promise() const {
        return *reinterpret_cast<Promise*>(
            static_cast<char*>(__frame_ptr_) + __promise_offset()
        );
    }

private:
    static constexpr size_t __promise_offset() {
        return sizeof(void*) * 2;  // resume + destroy 포인터 후
    }
};
```

### 컴파일러 변환 과정

```cpp
// 1. 원본 코드
Generator<int> count_up(int start, int end) {
    for (int i = start; i < end; ++i) {
        co_yield i;
    }
}

// 2. 컴파일러 변환 (의사 코드)
Generator<int> count_up(int start, int end) {
    // 프레임 할당
    __count_up_frame* __frame = new __count_up_frame();

    // 파라미터 복사
    __frame->start = start;
    __frame->end = end;

    // Promise 초기화
    new (&__frame->__promise) Generator<int>::promise_type();

    // 반환 객체 획득
    Generator<int> __return_object =
        __frame->__promise.get_return_object();

    // Resume/Destroy 함수 설정
    __frame->__resume_fn = &__count_up_resume;
    __frame->__destroy_fn = &__count_up_destroy;

    // Initial suspend 평가
    __frame->__initial_awaiter =
        __frame->__promise.initial_suspend();

    if (!__frame->__initial_awaiter.await_ready()) {
        __frame->__suspend_index = 0;
        __frame->__initial_awaiter.await_suspend(
            std::coroutine_handle<...>::from_promise(__frame->__promise)
        );
        return __return_object;  // 즉시 반환
    }

    // initial_suspend가 ready면 본문 시작
    __count_up_resume(__frame);

    return __return_object;
}

// Resume 함수 (상태 머신)
void __count_up_resume(__count_up_frame* __frame) {
    try {
        switch (__frame->__suspend_index) {
        case 0:  // initial_suspend 후
            goto __resume_point_0;
        case 1:  // yield 후
            goto __resume_point_1;
        }

    __resume_point_0:
        __frame->i = __frame->start;

    __loop_start:
        if (__frame->i >= __frame->end) {
            goto __final;
        }

        // co_yield i
        __frame->__promise.yield_value(__frame->i);
        __frame->__suspend_index = 1;
        return;  // 일시 중단

    __resume_point_1:
        ++__frame->i;
        goto __loop_start;

    __final:
        __frame->__promise.return_void();

    } catch (...) {
        __frame->__promise.unhandled_exception();
    }

    // final_suspend
    __frame->__final_awaiter =
        __frame->__promise.final_suspend();
    __frame->__suspend_index = 2;  // final
}
```

## 성능 고려사항

### 힙 할당 비용

```cpp
// 힙 할당을 줄이는 방법

// 1. 사용자 정의 allocator
template<typename T>
struct PoolAllocator {
    static void* allocate(size_t size) {
        return memory_pool.allocate(size);
    }

    static void deallocate(void* ptr, size_t size) {
        memory_pool.deallocate(ptr, size);
    }
};

template<typename T>
struct Generator<T>::promise_type {
    // 사용자 정의 할당
    void* operator new(size_t size) {
        return PoolAllocator<void>::allocate(size);
    }

    void operator delete(void* ptr, size_t size) {
        PoolAllocator<void>::deallocate(ptr, size);
    }
};

// 2. 스택 할당 (HALO 의존)
// 인라인 가능한 작은 coroutine 사용
```

### 벤치마크 비교

```cpp
#include <chrono>

void benchmark() {
    constexpr int N = 1000000;

    // 일반 함수 (baseline)
    auto start1 = std::chrono::high_resolution_clock::now();
    long sum1 = 0;
    for (int i = 0; i < N; ++i) {
        sum1 += i;
    }
    auto end1 = std::chrono::high_resolution_clock::now();

    // Generator coroutine
    auto start2 = std::chrono::high_resolution_clock::now();
    long sum2 = 0;
    for (int value : range(0, N)) {
        sum2 += value;
    }
    auto end2 = std::chrono::high_resolution_clock::now();

    auto dur1 = std::chrono::duration_cast<std::chrono::microseconds>(
        end1 - start1).count();
    auto dur2 = std::chrono::duration_cast<std::chrono::microseconds>(
        end2 - start2).count();

    std::cout << "일반 루프: " << dur1 << " μs\n";
    std::cout << "Generator: " << dur2 << " μs\n";
    std::cout << "오버헤드: " << (dur2 - dur1) * 100.0 / dur1 << "%\n";
}
```

## 실전 패턴

### 1. 비동기 I/O

```cpp
// 비동기 파일 읽기
Task<std::string> async_read_file(const std::string& path) {
    // OS 비동기 API 래핑
    auto file = co_await async_open(path);
    auto content = co_await file.read_all();
    co_await file.close();
    co_return content;
}

// 여러 파일 병렬 읽기
Task<std::vector<std::string>> read_all_files(
    const std::vector<std::string>& paths) {

    std::vector<Task<std::string>> tasks;
    for (const auto& path : paths) {
        tasks.push_back(async_read_file(path));
    }

    std::vector<std::string> results;
    for (auto& task : tasks) {
        results.push_back(co_await task);
    }

    co_return results;
}
```

### 2. 파이프라인 패턴

```cpp
// 데이터 변환 파이프라인
template<typename T, typename U>
Generator<U> transform(Generator<T> source,
                       std::function<U(T)> mapper) {
    for (auto& value : source) {
        co_yield mapper(std::move(value));
    }
}

template<typename T>
Generator<T> filter(Generator<T> source,
                    std::function<bool(const T&)> predicate) {
    for (auto& value : source) {
        if (predicate(value)) {
            co_yield std::move(value);
        }
    }
}

// 사용 예
auto pipeline =
    transform(
        filter(
            range(1, 100),
            [](int x) { return x % 2 == 0; }  // 짝수만
        ),
        [](int x) { return x * x; }  // 제곱
    );

for (int value : pipeline) {
    std::cout << value << " ";
}
// 출력: 4 16 36 64 100 ...
```

### 3. 상태 머신

```cpp
// 파서 상태 머신
enum class JsonToken {
    ObjectStart, ObjectEnd,
    ArrayStart, ArrayEnd,
    String, Number,
    Colon, Comma,
    True, False, Null,
    End
};

Generator<JsonToken> json_tokenizer(std::string_view input) {
    size_t pos = 0;

    while (pos < input.size()) {
        // 공백 스킵
        while (pos < input.size() && std::isspace(input[pos])) {
            ++pos;
        }

        if (pos >= input.size()) break;

        switch (input[pos]) {
        case '{': co_yield JsonToken::ObjectStart; ++pos; break;
        case '}': co_yield JsonToken::ObjectEnd; ++pos; break;
        case '[': co_yield JsonToken::ArrayStart; ++pos; break;
        case ']': co_yield JsonToken::ArrayEnd; ++pos; break;
        case ':': co_yield JsonToken::Colon; ++pos; break;
        case ',': co_yield JsonToken::Comma; ++pos; break;
        case '"':
            // 문자열 파싱...
            co_yield JsonToken::String;
            break;
        // ... 기타 토큰
        }
    }

    co_yield JsonToken::End;
}
```

## 요약

### 핵심 개념

| 개념 | 설명 |
|------|------|
| `co_await` | 비동기 작업 대기 및 일시 중단 |
| `co_yield` | 값 생성 및 일시 중단 (제너레이터) |
| `co_return` | coroutine 종료 |
| Promise Type | coroutine 동작 제어 |
| Coroutine Handle | coroutine 인스턴스 제어 |
| Awaiter | `co_await` 동작 정의 |

### 장점

- **메모리 효율**: 모든 값을 미리 계산하지 않음 (lazy evaluation)
- **표현력**: 복잡한 비동기 로직을 동기 코드처럼 작성
- **성능**: 스레드 없이 동시성 구현 가능
- **확장성**: 사용자 정의 Awaiter로 다양한 비동기 소스 지원

### 주의사항

- **학습 곡선**: Promise, Awaiter 개념 이해 필요
- **디버깅**: 상태 머신 변환으로 디버깅이 복잡해질 수 있음
- **힙 할당**: HALO 최적화가 적용되지 않으면 매 호출마다 힙 할당
- **예외 처리**: `unhandled_exception()` 반드시 구현 필요

## 참고 자료

- [cppreference: Coroutines (C++20)](https://en.cppreference.com/w/cpp/language/coroutines)
- [Lewis Baker - Asymmetric Transfer](https://lewissbaker.github.io/)
- [CppCon: C++ Coroutines 발표들](https://www.youtube.com/results?search_query=cppcon+coroutines)
- [libcoro](https://github.com/jbaldwin/libcoro) - 프로덕션 코루틴 라이브러리
