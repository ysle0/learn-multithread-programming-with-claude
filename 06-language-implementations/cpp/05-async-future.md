# C++의 Async와 Future

`std::async`와 `std::future`는 동시성에 대한 고수준 태스크 기반 접근 방식을 제공하여, 스레드를 관리하는 방법보다 무엇을 계산할지에 집중할 수 있게 합니다.

## 목차
- [기본 개념](#기본-개념)
- [std::async](#stdasync)
- [std::future](#stdfuture)
- [std::promise](#stdpromise)
- [std::packaged_task](#stdpackaged_task)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### std::async란?

`std::async`는 함수를 비동기적으로 실행하고 결과에 대한 `std::future`를 반환합니다:

```cpp
#include <future>
#include <iostream>

int compute() {
    return 42;
}

int main() {
    // 비동기 태스크 실행
    std::future<int> result = std::async(compute);

    // 다른 작업 수행...

    // 결과 가져오기 (준비되지 않았으면 블로킹)
    std::cout << "Result: " << result.get() << "\n";
    return 0;
}
```

### 태스크 기반 vs. 스레드 기반

```cpp
// 스레드 기반 (저수준)
std::thread t(compute);
t.join();

// 태스크 기반 (고수준)
auto future = std::async(compute);
auto result = future.get();
```

## std::async

### 실행 정책

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

    // 비동기 실행 보장 (새 스레드)
    auto f1 = std::async(std::launch::async, work);

    // 지연 실행 (get() 호출 시 실행)
    auto f2 = std::async(std::launch::deferred, work);

    // 구현이 선택 (기본값)
    auto f3 = std::async(work);

    std::cout << "Getting f1: " << f1.get() << "\n";
    std::cout << "Getting f2: " << f2.get() << "\n";
    std::cout << "Getting f3: " << f3.get() << "\n";

    return 0;
}
```

### 인수 전달

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
    // 함수에 인수 전달
    auto future1 = std::async(add, 5, 3);
    std::cout << "Sum: " << future1.get() << "\n";

    // 참조에 대한 참조 래퍼
    std::string msg = "Hello";
    auto future2 = std::async(print_message, std::cref(msg), 3);
    future2.wait();

    return 0;
}
```

### 람다 함수

```cpp
#include <future>
#include <iostream>

int main() {
    int x = 10;

    // 값으로 캡처
    auto f1 = std::async([x] {
        return x * 2;
    });

    // 참조로 캡처
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

### 멤버 함수

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

    // 비-const 멤버 함수
    auto f1 = std::async(&Calculator::multiply, &calc, 5, 3);
    std::cout << "Multiply: " << f1.get() << "\n";

    // const 멤버 함수
    auto f2 = std::async(&Calculator::add, &calc, 5, 3);
    std::cout << "Add: " << f2.get() << "\n";

    return 0;
}
```

### 예외 처리

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
        int result = future.get();  // 여기서 예외가 다시 던져짐
        std::cout << "Result: " << result << "\n";
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    return 0;
}
```

## std::future

### 기본 연산

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

    // 결과가 준비되었는지 확인
    while (future.wait_for(std::chrono::milliseconds(500))
           != std::future_status::ready) {
        std::cout << "Still waiting...\n";
    }

    // 결과 가져오기 (준비되지 않았으면 블로킹)
    std::cout << "Result: " << future.get() << "\n";

    // get()은 한 번만 호출 가능!
    // future.get();  // 정의되지 않은 동작

    return 0;
}
```

### wait()과 wait_for()

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

    // 결과를 가져오지 않고 대기
    future.wait();
    std::cout << "Computation finished\n";

    // 시간 제한 대기
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

### valid() 확인

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

    std::future<int> future2;  // 기본 생성
    std::cout << "future2 valid: " << future2.valid() << "\n";  // false

    return 0;
}
```

## std::promise

### 기본 사용법

```cpp
#include <future>
#include <thread>
#include <iostream>

void compute_value(std::promise<int> promise) {
    // 작업 수행
    std::this_thread::sleep_for(std::chrono::seconds(1));

    // 결과 설정
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

### 예외가 있는 Promise

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

### shared_future로 다중 대기자

```cpp
#include <future>
#include <thread>
#include <iostream>
#include <vector>

int main() {
    std::promise<int> promise;
    std::shared_future<int> shared_future = promise.get_future();

    // 여러 스레드가 shared_future를 대기할 수 있음
    std::vector<std::thread> threads;
    for (int i = 0; i < 3; ++i) {
        threads.emplace_back([shared_future, i] {
            std::cout << "Thread " << i << " got: "
                      << shared_future.get() << "\n";
        });
    }

    // 한 번만 값 설정
    promise.set_value(42);

    for (auto& t : threads) {
        t.join();
    }

    return 0;
}
```

## std::packaged_task

### 기본 사용법

```cpp
#include <future>
#include <thread>
#include <iostream>

int multiply(int a, int b) {
    return a * b;
}

int main() {
    // packaged_task 생성
    std::packaged_task<int(int, int)> task(multiply);

    // future 가져오기
    std::future<int> future = task.get_future();

    // 스레드에서 태스크 실행
    std::thread t(std::move(task), 5, 3);

    // 결과 가져오기
    std::cout << "Result: " << future.get() << "\n";

    t.join();
    return 0;
}
```

### 태스크 큐 예제

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

## 다른 언어와의 비교

### C++ vs. C#
```cpp
// C++
auto future = std::async([] { return 42; });
int result = future.get();

// C# 동등 코드:
// Task<int> task = Task.Run(() => 42);
// int result = await task;
```

### C++ vs. Go
```cpp
// C++ future
auto future = std::async(compute);
auto result = future.get();

// Go에는 future가 없음
// 대신 채널 사용:
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

## 모범 사례

### 1. 수동 스레드보다 std::async 선호

```cpp
// 좋음: 태스크 기반
auto future = std::async([] {
    return expensive_computation();
});
auto result = future.get();

// 덜 좋음: 스레드 기반 (더 많은 보일러플레이트)
int result;
std::thread t([&result] {
    result = expensive_computation();
});
t.join();
```

### 2. 필요한 경우 실행 정책 지정

```cpp
// 좋음: 명시적 async (새 스레드 보장)
auto future = std::async(std::launch::async, compute);

// 좋음: 지연 평가가 필요할 때 deferred
auto future = std::async(std::launch::deferred, compute);

// 괜찮음: 구현이 선택하도록 함 (기본값)
auto future = std::async(compute);
```

### 3. 반환된 Future를 무시하지 않기

```cpp
// 나쁨: future가 즉시 파괴되고, 소멸자에서 블로킹!
std::async(std::launch::async, [] {
    expensive_work();
});  // 여기서 블로킹!

// 좋음: 진정한 비동기를 원하면 future를 유지
auto future = std::async(std::launch::async, [] {
    expensive_work();
});
// 다른 작업 수행...
future.wait();
```

### 4. 다중 대기자에는 shared_future 사용

```cpp
// 좋음: 여러 스레드가 대기 가능
std::promise<int> promise;
std::shared_future<int> sf = promise.get_future();

std::thread t1([sf] { std::cout << sf.get() << "\n"; });
std::thread t2([sf] { std::cout << sf.get() << "\n"; });

promise.set_value(42);
t1.join();
t2.join();
```

### 5. 예외를 올바르게 처리

```cpp
// 좋음: 예외가 future를 통해 전파됨
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

## 일반적인 실수

### 1. Future 소멸자에서의 블로킹

```cpp
// 나쁨: async 정책이 사용된 경우 소멸자에서 블로킹!
{
    std::async(std::launch::async, long_running_task);
}  // 태스크를 기다리며 여기서 블로킹!

// 좋음: future를 유지하거나 deferred 사용
{
    auto future = std::async(std::launch::async, long_running_task);
    // 다른 작업 수행...
    future.wait();
}
```

### 2. get()을 여러 번 호출

```cpp
// 나쁨: get()은 한 번만 호출 가능
auto future = std::async(compute);
int r1 = future.get();  // OK
int r2 = future.get();  // 정의되지 않은 동작!

// 좋음: 결과를 저장
auto future = std::async(compute);
int result = future.get();
// result를 여러 번 사용
```

### 3. valid() 미확인

```cpp
// 나쁨: 유효하지 않은 future에 대한 연산
std::future<int> future;  // 기본 생성
int result = future.get();  // 정의되지 않은 동작!

// 좋음: 유효성 확인
if (future.valid()) {
    int result = future.get();
}
```

### 4. Deferred에서의 댕글링 참조

```cpp
// 나쁨: deferred 실행에서의 댕글링 참조
int compute_with_local() {
    int local_var = 42;
    auto future = std::async(std::launch::deferred, [&] {
        return local_var;  // 참조로 캡처
    });
    return future.get();  // OK, 아직 스코프 안에 있음
}

int bad_example() {
    int local_var = 42;
    auto future = std::async(std::launch::deferred, [&] {
        return local_var;  // 참조로 캡처
    });
    // future가 반환되고, local_var 파괴됨
    return 0;
}  // 나중에 get()이 파괴된 변수에 접근!

// 좋음: 값으로 캡처
auto future = std::async(std::launch::deferred, [local_var] {
    return local_var;
});
```

### 5. Promise에서의 경쟁 조건

```cpp
// 나쁨: 값이 검색되기 전에 promise가 파괴됨
std::future<int> bad_promise() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();
    promise.set_value(42);
    return future;  // promise가 파괴됨, 하지만 future는 여전히 유효? 타이밍에 따라 다름
}

// 좋음: promise가 충분히 오래 살도록 보장
std::future<int> good_promise() {
    auto promise = std::make_shared<std::promise<int>>();
    std::future<int> future = promise->get_future();

    std::thread([promise] {
        promise->set_value(42);
    }).detach();

    return future;
}
```

## 내부 메커니즘

### 공유 상태 아키텍처

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

### Future 소멸자 블로킹 문제

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

### Deferred 실행 메커니즘

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

### shared_future 복사 의미론

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

### wait_for 상태 감지

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

### packaged_task 내부 상태

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

## 성능 고려사항

### std::async의 오버헤드

```cpp
// std::async 오버헤드:
// - 스레드 생성 (launch::async인 경우): ~100 us
// - Future/promise 설정: ~1 us
// - get() 호출: 준비된 경우 무시할 수 있음

// 작은 태스크의 경우, 오버헤드가 작업보다 클 수 있음
auto f = std::async([] { return 1 + 1; });  // 오버헤드 >> 작업

// 100 us 이상 걸리는 태스크에 사용
auto f = std::async([] {
    return expensive_computation();  // OK
});
```

### 스레드 풀 대안

```cpp
// 작은 태스크가 많은 경우, 스레드 풀을 고려
// std::async는 너무 많은 스레드를 생성할 수 있음

// 더 좋음: 스레드 재사용
// (C++에는 내장 스레드 풀이 없지만,
//  packaged_task로 구현할 수 있음)
```

## 전체 예제: 병렬 계산

```cpp
#include <future>
#include <vector>
#include <iostream>
#include <numeric>
#include <algorithm>

// 범위의 합계 계산
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

    // 비동기 태스크 실행
    for (unsigned int i = 0; i < num_threads; ++i) {
        auto begin = data.begin() + i * chunk_size;
        auto end = (i == num_threads - 1) ? data.end()
                                          : begin + chunk_size;

        futures.push_back(std::async(std::launch::async,
                                    partial_sum, begin, end));
    }

    // 결과 수집
    long long total = 0;
    for (auto& future : futures) {
        total += future.get();
    }

    std::cout << "Total sum: " << total << "\n";
    return 0;
}
```

## 전체 예제: Future를 사용한 파이프라인

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

    // 각 입력에 대해 파이프라인 실행
    for (int input : inputs) {
        auto future = std::async(std::launch::async, [input] {
            int result = stage1(input);
            result = stage2(result);
            result = stage3(result);
            return result;
        });
        results.push_back(std::move(future));
    }

    // 결과 수집
    for (size_t i = 0; i < results.size(); ++i) {
        std::cout << "Input " << inputs[i]
                  << " -> Result: " << results[i].get() << "\n";
    }

    return 0;
}
```

## 추가 읽기

- [C++ Reference: std::async](https://en.cppreference.com/w/cpp/thread/async)
- [C++ Reference: std::future](https://en.cppreference.com/w/cpp/thread/future)
- [C++ Reference: std::promise](https://en.cppreference.com/w/cpp/thread/promise)
- [std::thread](./01-std-thread.md)

## 탐색

- [C++ 개요로 돌아가기](./README.md)
- 이전: [Condition Variable](./04-condition-variable.md)
- [언어 구현으로 돌아가기](../)
