# Async and Future in C++

`std::async` and `std::future` provide a high-level, task-based approach to concurrency, allowing you to focus on what to compute rather than how to manage threads.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [std::async](#stdasync)
- [std::future](#stdfuture)
- [std::promise](#stdpromise)
- [std::packaged_task](#stdpackaged_task)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is std::async?

`std::async` runs a function asynchronously and returns a `std::future` for the result:

```cpp
#include <future>
#include <iostream>

int compute() {
    return 42;
}

int main() {
    // Launch async task
    std::future<int> result = std::async(compute);

    // Do other work...

    // Get result (blocks if not ready)
    std::cout << "Result: " << result.get() << "\n";
    return 0;
}
```

### Task-Based vs. Thread-Based

```cpp
// Thread-based (low-level)
std::thread t(compute);
t.join();

// Task-based (high-level)
auto future = std::async(compute);
auto result = future.get();
```

## std::async

### Launch Policies

```cpp
#include <future>
#include <iostream>
#include <thread>

int work() {
    std::cout << "Thread ID: " << std::this_thread::get_id() << "\n";
    return 42;
}

int main() {
    std::cout << "Main thread ID: " << std::this_thread::get_id() << "\n";

    // Guaranteed async execution (new thread)
    auto f1 = std::async(std::launch::async, work);

    // Deferred execution (runs on get())
    auto f2 = std::async(std::launch::deferred, work);

    // Implementation chooses (default)
    auto f3 = std::async(work);

    std::cout << "Getting f1: " << f1.get() << "\n";
    std::cout << "Getting f2: " << f2.get() << "\n";
    std::cout << "Getting f3: " << f3.get() << "\n";

    return 0;
}
```

### Passing Arguments

```cpp
#include <future>
#include <iostream>
#include <string>

int add(int a, int b) {
    return a + b;
}

void print_message(const std::string& msg, int count) {
    for (int i = 0; i < count; ++i) {
        std::cout << msg << " " << i << "\n";
    }
}

int main() {
    // Arguments passed to function
    auto future1 = std::async(add, 5, 3);
    std::cout << "Sum: " << future1.get() << "\n";

    // Reference wrapper for references
    std::string msg = "Hello";
    auto future2 = std::async(print_message, std::cref(msg), 3);
    future2.wait();

    return 0;
}
```

### Lambda Functions

```cpp
#include <future>
#include <iostream>

int main() {
    int x = 10;

    // Capture by value
    auto f1 = std::async([x] {
        return x * 2;
    });

    // Capture by reference
    auto f2 = std::async([&x] {
        x += 5;
        return x;
    });

    std::cout << "f1: " << f1.get() << "\n";  // 20
    std::cout << "f2: " << f2.get() << "\n";  // 15
    std::cout << "x: " << x << "\n";           // 15

    return 0;
}
```

### Member Functions

```cpp
#include <future>
#include <iostream>

class Calculator {
public:
    int multiply(int a, int b) {
        return a * b;
    }

    int add(int a, int b) const {
        return a + b;
    }
};

int main() {
    Calculator calc;

    // Non-const member function
    auto f1 = std::async(&Calculator::multiply, &calc, 5, 3);
    std::cout << "Multiply: " << f1.get() << "\n";

    // Const member function
    auto f2 = std::async(&Calculator::add, &calc, 5, 3);
    std::cout << "Add: " << f2.get() << "\n";

    return 0;
}
```

### Exception Handling

```cpp
#include <future>
#include <iostream>
#include <stdexcept>

int may_throw(bool should_throw) {
    if (should_throw) {
        throw std::runtime_error("Error in async task");
    }
    return 42;
}

int main() {
    auto future = std::async(may_throw, true);

    try {
        int result = future.get();  // Exception re-thrown here
        std::cout << "Result: " << result << "\n";
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    return 0;
}
```

## std::future

### Basic Operations

```cpp
#include <future>
#include <iostream>
#include <thread>
#include <chrono>

int long_computation() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return 42;
}

int main() {
    auto future = std::async(std::launch::async, long_computation);

    // Check if result is ready
    while (future.wait_for(std::chrono::milliseconds(500))
           != std::future_status::ready) {
        std::cout << "Still waiting...\n";
    }

    // Get result (blocks if not ready)
    std::cout << "Result: " << future.get() << "\n";

    // Can only call get() once!
    // future.get();  // Undefined behavior

    return 0;
}
```

### wait() and wait_for()

```cpp
#include <future>
#include <iostream>
#include <chrono>

int compute() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    return 42;
}

int main() {
    auto future = std::async(std::launch::async, compute);

    // Wait without getting result
    future.wait();
    std::cout << "Computation finished\n";

    // Timed wait
    auto status = future.wait_for(std::chrono::milliseconds(100));

    if (status == std::future_status::ready) {
        std::cout << "Result: " << future.get() << "\n";
    } else if (status == std::future_status::timeout) {
        std::cout << "Timeout\n";
    } else {  // std::future_status::deferred
        std::cout << "Deferred\n";
    }

    return 0;
}
```

### valid() Check

```cpp
#include <future>
#include <iostream>

int compute() {
    return 42;
}

int main() {
    std::future<int> future1 = std::async(compute);
    std::cout << "future1 valid: " << future1.valid() << "\n";  // true

    int result = future1.get();
    std::cout << "future1 valid: " << future1.valid() << "\n";  // false

    std::future<int> future2;  // Default constructed
    std::cout << "future2 valid: " << future2.valid() << "\n";  // false

    return 0;
}
```

## std::promise

### Basic Usage

```cpp
#include <future>
#include <thread>
#include <iostream>

void compute_value(std::promise<int> promise) {
    // Do some work
    std::this_thread::sleep_for(std::chrono::seconds(1));

    // Set the result
    promise.set_value(42);
}

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::thread t(compute_value, std::move(promise));

    std::cout << "Waiting for result...\n";
    std::cout << "Result: " << future.get() << "\n";

    t.join();
    return 0;
}
```

### Promise with Exception

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <stdexcept>

void compute_with_error(std::promise<int> promise, bool error) {
    try {
        if (error) {
            throw std::runtime_error("Computation failed");
        }
        promise.set_value(42);
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::thread t(compute_with_error, std::move(promise), true);

    try {
        std::cout << "Result: " << future.get() << "\n";
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    t.join();
    return 0;
}
```

### Multiple Waiters with shared_future

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <vector>

int main() {
    std::promise<int> promise;
    std::shared_future<int> shared_future = promise.get_future();

    // Multiple threads can wait on shared_future
    std::vector<std::thread> threads;
    for (int i = 0; i < 3; ++i) {
        threads.emplace_back([shared_future, i] {
            std::cout << "Thread " << i << " got: "
                      << shared_future.get() << "\n";
        });
    }

    // Set value once
    promise.set_value(42);

    for (auto& t : threads) {
        t.join();
    }

    return 0;
}
```

## std::packaged_task

### Basic Usage

```cpp
#include <future>
#include <thread>
#include <iostream>

int multiply(int a, int b) {
    return a * b;
}

int main() {
    // Create packaged task
    std::packaged_task<int(int, int)> task(multiply);

    // Get future
    std::future<int> future = task.get_future();

    // Run task in thread
    std::thread t(std::move(task), 5, 3);

    // Get result
    std::cout << "Result: " << future.get() << "\n";

    t.join();
    return 0;
}
```

### Task Queue Example

```cpp
#include <future>
#include <queue>
#include <thread>
#include <iostream>
#include <mutex>
#include <condition_variable>

class TaskQueue {
    std::queue<std::packaged_task<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool done = false;

public:
    template<typename Func>
    std::future<void> enqueue(Func func) {
        std::packaged_task<void()> task(func);
        std::future<void> future = task.get_future();

        {
            std::lock_guard<std::mutex> lock(mtx);
            tasks.push(std::move(task));
        }
        cv.notify_one();

        return future;
    }

    void worker() {
        while (true) {
            std::packaged_task<void()> task;
            {
                std::unique_lock<std::mutex> lock(mtx);
                cv.wait(lock, [this] { return !tasks.empty() || done; });

                if (done && tasks.empty()) {
                    return;
                }

                task = std::move(tasks.front());
                tasks.pop();
            }
            task();
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mtx);
            done = true;
        }
        cv.notify_all();
    }
};

int main() {
    TaskQueue queue;
    std::thread worker(&TaskQueue::worker, &queue);

    auto f1 = queue.enqueue([] { std::cout << "Task 1\n"; });
    auto f2 = queue.enqueue([] { std::cout << "Task 2\n"; });
    auto f3 = queue.enqueue([] { std::cout << "Task 3\n"; });

    f1.wait();
    f2.wait();
    f3.wait();

    queue.stop();
    worker.join();

    return 0;
}
```

## Comparison with Other Languages

### C++ vs. C#
```cpp
// C++
auto future = std::async([] { return 42; });
int result = future.get();

// C# equivalent:
// Task<int> task = Task.Run(() => 42);
// int result = await task;
```

### C++ vs. Go
```cpp
// C++ future
auto future = std::async(compute);
auto result = future.get();

// Go doesn't have futures
// Use channels instead:
// ch := make(chan int)
// go func() { ch <- compute() }()
// result := <-ch
```

### C++ vs. JavaScript
```cpp
// C++ future
auto future = std::async(compute);
auto result = future.get();

// JavaScript Promise:
// const promise = new Promise((resolve) => {
//     resolve(compute());
// });
// const result = await promise;
```

## Best Practices

### 1. Prefer std::async Over Manual Threads

```cpp
// GOOD: Task-based
auto future = std::async([] {
    return expensive_computation();
});
auto result = future.get();

// LESS GOOD: Thread-based (more boilerplate)
int result;
std::thread t([&result] {
    result = expensive_computation();
});
t.join();
```

### 2. Specify Launch Policy When Needed

```cpp
// GOOD: Explicit async (guaranteed new thread)
auto future = std::async(std::launch::async, compute);

// GOOD: Deferred when you want lazy evaluation
auto future = std::async(std::launch::deferred, compute);

// OK: Let implementation choose (default)
auto future = std::async(compute);
```

### 3. Don't Ignore Returned Futures

```cpp
// BAD: Future destroyed immediately, blocks in destructor!
std::async(std::launch::async, [] {
    expensive_work();
});  // Blocks here!

// GOOD: Keep future if you want true async
auto future = std::async(std::launch::async, [] {
    expensive_work();
});
// Do other work...
future.wait();
```

### 4. Use shared_future for Multiple Waiters

```cpp
// GOOD: Multiple threads can wait
std::promise<int> promise;
std::shared_future<int> sf = promise.get_future();

std::thread t1([sf] { std::cout << sf.get() << "\n"; });
std::thread t2([sf] { std::cout << sf.get() << "\n"; });

promise.set_value(42);
t1.join();
t2.join();
```

### 5. Handle Exceptions Properly

```cpp
// GOOD: Exceptions propagated through future
auto future = std::async([] {
    if (error_condition) {
        throw std::runtime_error("Error");
    }
    return 42;
});

try {
    auto result = future.get();
} catch (const std::exception& e) {
    std::cerr << "Error: " << e.what() << "\n";
}
```

## Common Pitfalls

### 1. Blocking in Future Destructor

```cpp
// BAD: Blocks in destructor if async policy used!
{
    std::async(std::launch::async, long_running_task);
}  // Blocks here waiting for task!

// GOOD: Keep future alive or use deferred
{
    auto future = std::async(std::launch::async, long_running_task);
    // Do other work...
    future.wait();
}
```

### 2. Calling get() Multiple Times

```cpp
// BAD: Can only call get() once
auto future = std::async(compute);
int r1 = future.get();  // OK
int r2 = future.get();  // Undefined behavior!

// GOOD: Store result
auto future = std::async(compute);
int result = future.get();
// Use result multiple times
```

### 3. Not Checking valid()

```cpp
// BAD: Operating on invalid future
std::future<int> future;  // Default constructed
int result = future.get();  // Undefined behavior!

// GOOD: Check validity
if (future.valid()) {
    int result = future.get();
}
```

### 4. Dangling References with Deferred

```cpp
// BAD: Dangling reference with deferred execution
int compute_with_local() {
    int local_var = 42;
    auto future = std::async(std::launch::deferred, [&] {
        return local_var;  // Captures by reference
    });
    return future.get();  // OK, still in scope
}

int bad_example() {
    int local_var = 42;
    auto future = std::async(std::launch::deferred, [&] {
        return local_var;  // Captures by reference
    });
    // future returned, local_var destroyed
    return 0;
}  // Later get() will access destroyed variable!

// GOOD: Capture by value
auto future = std::async(std::launch::deferred, [local_var] {
    return local_var;
});
```

### 5. Race Condition with Promise

```cpp
// BAD: Promise destroyed before value retrieved
std::future<int> bad_promise() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();
    promise.set_value(42);
    return future;  // Promise destroyed, but future still valid? Depends on timing
}

// GOOD: Ensure promise lives long enough
std::future<int> good_promise() {
    auto promise = std::make_shared<std::promise<int>>();
    std::future<int> future = promise->get_future();

    std::thread([promise] {
        promise->set_value(42);
    }).detach();

    return future;
}
```

## Internal Mechanisms

### Shared State Architecture

`std::future`와 `std::promise`는 공유 상태(Shared State)를 통해 통신합니다:

```cpp
// 공유 상태 구조 (libstdc++ 단순화)
struct __future_base::_Result<T> {
    T _M_value;                    // 결과값 저장
    exception_ptr _M_error;        // 예외 저장
};

struct __future_base::_State_baseV2 {
    _Result_base* _M_result;       // 결과 (값 또는 예외)
    atomic<bool> _M_retrieved;     // get() 호출 여부
    atomic<int> _M_ready;          // 결과 준비 완료 플래그

    mutex _M_mutex;
    condition_variable _M_cond;    // 대기자 깨우기용

    // 결과 설정 (promise.set_value)
    void _M_set_result(...) {
        lock_guard lk(_M_mutex);
        _M_result = ...;
        _M_ready = true;
        _M_cond.notify_all();      // 대기자 깨움
    }

    // 결과 대기 (future.get)
    _Result_base* _M_get_result() {
        unique_lock lk(_M_mutex);
        _M_cond.wait(lk, [this]{ return _M_ready; });
        _M_retrieved = true;
        return _M_result;
    }
};
```

**메모리 레이아웃**:
```
┌─────────────────────────────────────────────────────────────┐
│                     Shared State                            │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  atomic<bool> ready                                  │  │
│  │  atomic<int> ref_count                              │  │
│  │  mutex                                               │  │
│  │  condition_variable                                  │  │
│  │  ┌─────────────────────────────────────────────┐   │  │
│  │  │  Result<T>                                   │   │  │
│  │  │    T value  OR  exception_ptr error         │   │  │
│  │  └─────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
          ▲                                    ▲
          │                                    │
    ┌─────┴─────┐                       ┌─────┴─────┐
    │  promise  │                       │   future  │
    │  (writer) │                       │  (reader) │
    └───────────┘                       └───────────┘
```

### Launch Policy 구현

```cpp
// std::launch 정책
enum class launch {
    async    = 1,   // 새 스레드에서 즉시 실행
    deferred = 2    // get() 호출 시 현재 스레드에서 실행
};

// std::async 내부 구현
template<typename F, typename... Args>
future<result_type> async(launch policy, F&& f, Args&&... args) {
    if (policy & launch::async) {
        // 새 스레드 생성하여 실행
        auto state = make_shared<__async_state<result_type>>();

        thread t([state, f = forward<F>(f), args...]() mutable {
            try {
                if constexpr (is_void_v<result_type>) {
                    invoke(f, args...);
                    state->set_value();
                } else {
                    state->set_value(invoke(f, args...));
                }
            } catch (...) {
                state->set_exception(current_exception());
            }
        });

        // 중요: 스레드를 state에 저장하여 future 소멸 시 join
        state->_M_thread = move(t);
        return future<result_type>(state);
    }
    else if (policy & launch::deferred) {
        // 함수와 인수를 저장만 함
        auto state = make_shared<__deferred_state<result_type>>(
            forward<F>(f), forward<Args>(args)...);
        return future<result_type>(state);
    }
}
```

### Future Destructor Blocking Issue

```cpp
// async(launch::async)로 생성된 future의 소멸자는 블로킹!
{
    auto f = std::async(std::launch::async, []{ sleep(10s); });
}  // <-- 여기서 10초 대기!

// 이유: async가 반환한 future는 특별한 "async state"를 가짐
// 소멸 시 스레드가 완료될 때까지 join()
```

**libstdc++ __async_state**:
```cpp
struct __async_state : __future_base::_State_baseV2 {
    thread _M_thread;

    ~__async_state() {
        if (_M_thread.joinable()) {
            _M_thread.join();  // 블로킹 소멸자
        }
    }
};
```

**해결 방법**:
```cpp
// 1. future를 저장하고 나중에 처리
auto future = std::async(std::launch::async, work);
// ... 다른 작업 ...
future.wait();

// 2. deferred 사용 (블로킹 없음)
auto future = std::async(std::launch::deferred, work);

// 3. 명시적으로 분리 (fire-and-forget)
std::thread([]{ long_running_work(); }).detach();
// 주의: 예외 전파 없음, 결과 받을 수 없음
```

### Deferred Execution Mechanism

```cpp
// deferred 상태: 함수와 인수를 저장
template<typename R>
struct __deferred_state : __future_base::_State_baseV2 {
    packaged_task<R()> _M_task;  // 지연 실행할 작업
    bool _M_executed = false;

    // get() 또는 wait() 호출 시 실행
    void _M_run() {
        if (!_M_executed) {
            _M_executed = true;
            _M_task();  // 현재 스레드에서 실행
        }
    }
};

// future::get() 에서:
T get() {
    if (_M_state->_M_is_deferred()) {
        _M_state->_M_run();  // 지금 실행!
    }
    return _M_state->_M_get_result()->_M_value;
}
```

### shared_future Copy Semantics

```cpp
// future는 이동만 가능, shared_future는 복사 가능
class shared_future<T> {
    shared_ptr<__state_type> _M_state;  // 공유 포인터

public:
    shared_future(const shared_future& other) noexcept
        : _M_state(other._M_state) {}  // 참조 카운트 증가

    // get()은 참조 반환 (복사 안 함)
    const T& get() const {
        return _M_state->_M_result->_M_value;
    }
};

// future에서 변환
future<int> f = async([] { return 42; });
shared_future<int> sf = f.share();  // future 무효화
// 이후 sf 복사 가능
```

### wait_for Status Detection

```cpp
// wait_for의 반환값으로 상태 확인
enum class future_status {
    ready,     // 결과 준비됨
    timeout,   // 시간 초과
    deferred   // deferred 정책으로 생성됨 (아직 실행 안 됨)
};

// deferred 감지
auto f = async(launch::deferred, work);
if (f.wait_for(0s) == future_status::deferred) {
    // get()을 호출할 때까지 실행되지 않음
}

// 무한 대기 루프 주의
// deferred future에 wait_for를 반복 호출하면 영원히 timeout
while (f.wait_for(100ms) == future_status::timeout) {
    // deferred면 여기서 무한 루프!
}
```

### packaged_task Internal State

```cpp
// packaged_task = callable + shared_state
template<typename R, typename... Args>
class packaged_task<R(Args...)> {
    function<R(Args...)> _M_fn;           // 저장된 callable
    shared_ptr<__state_type> _M_state;    // 공유 상태

public:
    void operator()(Args... args) {
        try {
            _M_state->set_value(_M_fn(forward<Args>(args)...));
        } catch (...) {
            _M_state->set_exception(current_exception());
        }
    }

    future<R> get_future() {
        return future<R>(_M_state);
    }

    // reset(): 새 공유 상태 생성 (재사용 가능)
    void reset() {
        _M_state = make_shared<__state_type>();
    }
};
```

## Performance Considerations

### Overhead of std::async

```cpp
// std::async overhead:
// - Thread creation (if launch::async): ~100 μs
// - Future/promise setup: ~1 μs
// - get() call: negligible if ready

// For small tasks, overhead may dominate
auto f = std::async([] { return 1 + 1; });  // Overhead >> work

// Use for tasks that take > 100 μs
auto f = std::async([] {
    return expensive_computation();  // OK
});
```

### Thread Pool Alternative

```cpp
// For many small tasks, consider thread pool
// std::async may create too many threads

// Better: Reuse threads
// (C++ doesn't have built-in thread pool,
//  but you can implement one with packaged_task)
```

## Complete Example: Parallel Computation

```cpp
#include <future>
#include <vector>
#include <iostream>
#include <numeric>
#include <algorithm>

// Compute sum of range
long long partial_sum(std::vector<int>::iterator begin,
                     std::vector<int>::iterator end) {
    return std::accumulate(begin, end, 0LL);
}

int main() {
    const size_t data_size = 10'000'000;
    std::vector<int> data(data_size, 1);

    const unsigned int num_threads = std::thread::hardware_concurrency();
    std::vector<std::future<long long>> futures;

    size_t chunk_size = data_size / num_threads;

    // Launch async tasks
    for (unsigned int i = 0; i < num_threads; ++i) {
        auto begin = data.begin() + i * chunk_size;
        auto end = (i == num_threads - 1) ? data.end()
                                          : begin + chunk_size;

        futures.push_back(std::async(std::launch::async,
                                    partial_sum, begin, end));
    }

    // Collect results
    long long total = 0;
    for (auto& future : futures) {
        total += future.get();
    }

    std::cout << "Total sum: " << total << "\n";
    return 0;
}
```

## Complete Example: Pipeline with Futures

```cpp
#include <future>
#include <iostream>
#include <vector>

int stage1(int input) {
    return input * 2;
}

int stage2(int input) {
    return input + 10;
}

int stage3(int input) {
    return input * input;
}

int main() {
    std::vector<int> inputs = {1, 2, 3, 4, 5};
    std::vector<std::future<int>> results;

    // Launch pipeline for each input
    for (int input : inputs) {
        auto future = std::async(std::launch::async, [input] {
            int result = stage1(input);
            result = stage2(result);
            result = stage3(result);
            return result;
        });
        results.push_back(std::move(future));
    }

    // Collect results
    for (size_t i = 0; i < results.size(); ++i) {
        std::cout << "Input " << inputs[i]
                  << " -> Result: " << results[i].get() << "\n";
    }

    return 0;
}
```

## Further Reading

- [C++ Reference: std::async](https://en.cppreference.com/w/cpp/thread/async)
- [C++ Reference: std::future](https://en.cppreference.com/w/cpp/thread/future)
- [C++ Reference: std::promise](https://en.cppreference.com/w/cpp/thread/promise)
- [std::thread](./01-std-thread.md)

## Navigation

- [Back to C++ Overview](./README.md)
- Previous: [Condition Variables](./04-condition-variable.md)
- [Back to Language Implementations](../)
