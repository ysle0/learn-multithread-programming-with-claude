# Future/Promise 패턴

## 개요

Future/Promise 패턴은 미래에 사용 가능해질 값을 나타냅니다. Promise는 값을 설정하는 쓰기 쪽이고, Future는 값을 가져오는 읽기 쪽입니다. 이 패턴은 비동기 프로그래밍에 필수적이며, 깔끔하게 합성 가능한 논블로킹 코드를 작성할 수 있게 해줍니다.

## 문제 정의

Future 없이 비동기 프로그래밍을 하면 다음과 같은 문제가 발생합니다:
- 콜백 지옥 (깊게 중첩된 콜백)
- 어려운 에러 처리
- 비동기 연산의 합성이 어려움
- 스레드 동기화 복잡성
- 보류 중인 결과를 표현하는 표준 방법의 부재

## 솔루션 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  Promise (Writer)           Future (Reader)         │
│       │                          │                  │
│       │   ┌──────────────┐      │                  │
│       └──▶│ Shared State │◀─────┘                  │
│           │              │                          │
│           │  - Value     │                          │
│           │  - Exception │                          │
│           │  - Ready?    │                          │
│           └──────────────┘                          │
│                                                     │
│  set_value() ───────────────────▶ get()            │
│  set_exception() ────────────────▶ wait()          │
│                                    then()           │
│                                                     │
└─────────────────────────────────────────────────────┘

타임라인:
1. promise/future 쌍 생성
2. consumer에게 future 전달
3. promise로 비동기 연산 시작
4. Consumer가 future를 대기하거나 콜백 등록
5. Producer가 promise에 값/예외 설정
6. Future가 ready 상태가 됨
7. Consumer가 값 또는 예외를 가져옴
```

## 기본 구현 (std::future 사용)

### 간단한 비동기 연산

```cpp
#include <future>
#include <iostream>
#include <thread>
#include <chrono>

int expensive_computation(int x) {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    return x * x;
}

int main() {
    // 비동기 태스크 실행
    std::future<int> future = std::async(std::launch::async,
                                         expensive_computation, 42);

    std::cout << "Computation started, doing other work...\n";

    // 연산이 실행되는 동안 다른 작업 수행
    std::this_thread::sleep_for(std::chrono::seconds(1));

    std::cout << "Getting result...\n";
    int result = future.get();  // ready 될 때까지 블로킹

    std::cout << "Result: " << result << "\n";

    return 0;
}
```

### Promise/Future 쌍

```cpp
#include <future>
#include <thread>

void producer(std::promise<int> promise) {
    std::this_thread::sleep_for(std::chrono::seconds(1));

    try {
        int result = 42;  // 무언가를 계산
        promise.set_value(result);  // promise 이행
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::thread t(producer, std::move(promise));

    std::cout << "Waiting for result...\n";
    int result = future.get();  // 값이 설정될 때까지 블로킹

    std::cout << "Result: " << result << "\n";

    t.join();
    return 0;
}
```

## 내부 메커니즘

### std::future의 내부 구조

std::promise/future 쌍은 공유 상태(Shared State)를 통해 통신합니다.

```cpp
// libstdc++ shared state 구조 (간략화)
template<typename T>
struct __shared_state {
    // 상태 플래그
    enum State {
        NOT_READY,
        READY,
        EXCEPTION
    };
    std::atomic<State> _state{NOT_READY};

    // 저장된 값 또는 예외
    union {
        T _value;
        std::exception_ptr _exception;
    };

    // 대기 동기화
    std::mutex _mutex;
    std::condition_variable _cv;

    // 연속(continuation) 지원 (C++20)
    std::function<void()> _continuation;

    // 참조 카운트
    std::atomic<int> _ref_count{2};  // promise + future

    // 값 설정 (promise.set_value)
    void set_value(T val) {
        std::lock_guard lock(_mutex);
        if (_state != NOT_READY) {
            throw std::future_error(future_errc::promise_already_satisfied);
        }
        new (&_value) T(std::move(val));
        _state.store(READY, std::memory_order_release);
        _cv.notify_all();

        if (_continuation) {
            _continuation();
        }
    }

    // 값 획득 (future.get)
    T get_value() {
        std::unique_lock lock(_mutex);
        _cv.wait(lock, [this] {
            return _state.load(std::memory_order_acquire) != NOT_READY;
        });

        if (_state == EXCEPTION) {
            std::rethrow_exception(_exception);
        }

        return std::move(_value);
    }
};
```

### 메모리 레이아웃

```
┌─────────────────────────────────────────────────────────────┐
│                Future/Promise Memory Layout                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  std::promise<T>                std::future<T>              │
│  ┌─────────────────┐            ┌─────────────────┐         │
│  │ shared_state*   │────┐  ┌────│ shared_state*   │         │
│  └─────────────────┘    │  │    └─────────────────┘         │
│                         │  │                                │
│                         ▼  ▼                                │
│                   ┌──────────────┐                          │
│                   │ Shared State │  (힙에 할당)              │
│                   │              │                          │
│                   │ ref_count: 2 │                          │
│                   │ state: ...   │                          │
│                   │ value/exc    │                          │
│                   │ mutex        │                          │
│                   │ cv           │                          │
│                   └──────────────┘                          │
│                                                             │
│  Promise 소멸 시: ref_count-- (1로)                         │
│  Future 소멸 시: ref_count-- (0이면 delete)                 │
└─────────────────────────────────────────────────────────────┘
```

### std::async의 동작 모드

```cpp
/*
 * std::launch::async:
 *   - 새 스레드에서 즉시 실행
 *   - std::thread와 유사하지만 future 반환
 *
 * std::launch::deferred:
 *   - future.get() 또는 wait() 호출 시점에 실행
 *   - 호출한 스레드에서 실행 (새 스레드 없음)
 *   - "lazy evaluation"
 *
 * std::launch::async | std::launch::deferred (기본값):
 *   - 구현체가 선택
 *   - 시스템 부하에 따라 결정
 */

// 내부 구현 개념
template<typename F, typename... Args>
auto async(std::launch policy, F&& f, Args&&... args) {
    using R = std::invoke_result_t<F, Args...>;

    if (policy & std::launch::async) {
        // 새 스레드 생성
        std::promise<R> promise;
        std::future<R> future = promise.get_future();

        std::thread([promise = std::move(promise),
                     f = std::forward<F>(f),
                     ...args = std::forward<Args>(args)]() mutable {
            try {
                if constexpr (std::is_void_v<R>) {
                    f(args...);
                    promise.set_value();
                } else {
                    promise.set_value(f(args...));
                }
            } catch (...) {
                promise.set_exception(std::current_exception());
            }
        }).detach();  // 또는 future 소멸자에서 join

        return future;
    }
    else if (policy & std::launch::deferred) {
        // 함수와 인자를 저장, 나중에 실행
        return deferred_future<R>(
            [f = std::forward<F>(f), ...args = std::forward<Args>(args)]() {
                return f(args...);
            });
    }
}
```

### Future의 블로킹 대기 구현

```c
// future.get()이 호출될 때의 커널 수준 동작

// 1. User space: condition_variable::wait()
void wait() {
    std::unique_lock lock(_mutex);
    _cv.wait(lock, [this] { return _state != NOT_READY; });
}

// 2. pthread_cond_wait → futex 시스템 콜
// futex(&_cv.__data.__wseq, FUTEX_WAIT, expected, NULL);

// 3. 커널: 스레드를 대기 큐에 추가
//    스케줄러가 다른 스레드 실행

// 4. promise.set_value() 호출 시
//    → pthread_cond_signal() → futex(FUTEX_WAKE, 1)
//    → 커널이 대기 스레드 깨움

// 5. 스레드가 런큐에 추가, 스케줄링되어 실행 재개
```

### std::packaged_task의 구조

```cpp
// packaged_task는 callable을 래핑하고 future를 제공
template<typename R, typename... Args>
class packaged_task<R(Args...)> {
    // 내부적으로 shared_state와 callable 보유
    std::shared_ptr<__shared_state<R>> _state;
    std::function<R(Args...)> _func;

public:
    packaged_task(std::function<R(Args...)> f)
        : _state(std::make_shared<__shared_state<R>>()),
          _func(std::move(f)) {}

    std::future<R> get_future() {
        return std::future<R>(_state);
    }

    void operator()(Args... args) {
        try {
            if constexpr (std::is_void_v<R>) {
                _func(args...);
                _state->set_value();
            } else {
                _state->set_value(_func(args...));
            }
        } catch (...) {
            _state->set_exception(std::current_exception());
        }
    }
};

/*
 * 사용 패턴:
 *
 * 1. packaged_task 생성
 * 2. get_future()로 future 획득
 * 3. task를 다른 스레드로 전달 (move)
 * 4. 다른 스레드에서 task() 호출
 * 5. 원래 스레드에서 future.get()
 *
 * Thread Pool과 함께 자주 사용됨
 */
```

### Continuation (then) 구현 원리

```cpp
// C++20 이전의 then() 구현 패턴
template<typename T>
template<typename F>
auto Future<T>::then(F&& func) -> Future<std::invoke_result_t<F, T>> {
    using R = std::invoke_result_t<F, T>;

    Promise<R> next_promise;
    Future<R> next_future = next_promise.get_future();

    std::unique_lock lock(_state->mutex);

    if (_state->ready) {
        // 이미 ready: 즉시 실행
        lock.unlock();
        try {
            if constexpr (std::is_void_v<T>) {
                next_promise.set_value(func());
            } else {
                next_promise.set_value(func(_state->get_value()));
            }
        } catch (...) {
            next_promise.set_exception(std::current_exception());
        }
    } else {
        // 아직 not ready: continuation 등록
        _state->continuation = [func = std::forward<F>(func),
                                promise = std::move(next_promise),
                                state = _state]() mutable {
            try {
                promise.set_value(func(state->get_value()));
            } catch (...) {
                promise.set_exception(std::current_exception());
            }
        };
    }

    return next_future;
}

/*
 * Continuation 체인:
 *
 * future1 → then(f1) → future2 → then(f2) → future3
 *
 * future1 완료 시:
 *   1. f1 실행 → future2 완료
 *   2. f2 실행 → future3 완료
 *   3. 최종 결과 사용 가능
 */
```

## 고급 구현: 합성 가능한 Future

### Continuation을 지원하는 커스텀 Future

```cpp
#include <memory>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <optional>
#include <exception>
#include <variant>

template<typename T>
class Future;

template<typename T>
class Promise {
private:
    struct SharedState {
        std::mutex mutex;
        std::condition_variable cv;
        std::variant<std::monostate, T, std::exception_ptr> value;
        std::function<void(const T&)> continuation;
        bool ready = false;

        void set(T val) {
            std::unique_lock<std::mutex> lock(mutex);

            if (ready) {
                throw std::runtime_error("Promise already fulfilled");
            }

            value = std::move(val);
            ready = true;

            if (continuation) {
                auto cont = std::move(continuation);
                lock.unlock();
                cont(std::get<T>(value));
            } else {
                cv.notify_all();
            }
        }

        void set_exception(std::exception_ptr ex) {
            std::unique_lock<std::mutex> lock(mutex);

            if (ready) {
                throw std::runtime_error("Promise already fulfilled");
            }

            value = ex;
            ready = true;
            cv.notify_all();
        }
    };

    std::shared_ptr<SharedState> state_;

public:
    Promise() : state_(std::make_shared<SharedState>()) {}

    Future<T> get_future() {
        return Future<T>(state_);
    }

    void set_value(T value) {
        state_->set(std::move(value));
    }

    void set_exception(std::exception_ptr ex) {
        state_->set_exception(ex);
    }
};

template<typename T>
class Future {
private:
    std::shared_ptr<typename Promise<T>::SharedState> state_;

public:
    Future(std::shared_ptr<typename Promise<T>::SharedState> state)
        : state_(state) {}

    // 블로킹 get
    T get() {
        std::unique_lock<std::mutex> lock(state_->mutex);

        state_->cv.wait(lock, [this] { return state_->ready; });

        if (std::holds_alternative<T>(state_->value)) {
            return std::get<T>(state_->value);
        } else if (std::holds_alternative<std::exception_ptr>(state_->value)) {
            std::rethrow_exception(std::get<std::exception_ptr>(state_->value));
        }

        throw std::runtime_error("Future not ready");
    }

    // 논블로킹 확인
    bool is_ready() const {
        std::lock_guard<std::mutex> lock(state_->mutex);
        return state_->ready;
    }

    // 값을 가져오지 않고 대기
    void wait() const {
        std::unique_lock<std::mutex> lock(state_->mutex);
        state_->cv.wait(lock, [this] { return state_->ready; });
    }

    // 타임아웃 대기
    template<typename Rep, typename Period>
    bool wait_for(const std::chrono::duration<Rep, Period>& timeout) const {
        std::unique_lock<std::mutex> lock(state_->mutex);
        return state_->cv.wait_for(lock, timeout,
            [this] { return state_->ready; });
    }

    // Continuation (then)
    template<typename F>
    auto then(F&& func) -> Future<decltype(func(std::declval<T>()))> {
        using ResultType = decltype(func(std::declval<T>()));

        Promise<ResultType> promise;
        auto future = promise.get_future();

        std::unique_lock<std::mutex> lock(state_->mutex);

        if (state_->ready) {
            // 이미 ready, 즉시 실행
            lock.unlock();

            try {
                if (std::holds_alternative<T>(state_->value)) {
                    promise.set_value(func(std::get<T>(state_->value)));
                } else {
                    promise.set_exception(std::get<std::exception_ptr>(state_->value));
                }
            } catch (...) {
                promise.set_exception(std::current_exception());
            }
        } else {
            // 아직 not ready, continuation 등록
            state_->continuation = [func = std::forward<F>(func),
                                    promise = std::move(promise)](const T& value) mutable {
                try {
                    promise.set_value(func(value));
                } catch (...) {
                    promise.set_exception(std::current_exception());
                }
            };
        }

        return future;
    }
};
```

### 사용 예제: 비동기 연산 체이닝

```cpp
Promise<int> promise;
auto future = promise.get_future();

// 여러 연산을 체이닝
auto result = future
    .then([](int x) {
        std::cout << "Step 1: " << x << "\n";
        return x * 2;
    })
    .then([](int x) {
        std::cout << "Step 2: " << x << "\n";
        return x + 10;
    })
    .then([](int x) {
        std::cout << "Step 3: " << x << "\n";
        return std::to_string(x);
    });

// 다른 스레드에서 promise 이행
std::thread t([promise = std::move(promise)]() mutable {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    promise.set_value(5);
});

std::string final_result = result.get();
std::cout << "Final: " << final_result << "\n";
// 출력: "20"

t.join();
```

## 병렬 실행 패턴

### All Of (모두 대기)

```cpp
template<typename T>
Future<std::vector<T>> when_all(std::vector<Future<T>> futures) {
    Promise<std::vector<T>> promise;
    auto result_future = promise.get_future();

    auto state = std::make_shared<struct {
        std::mutex mutex;
        std::vector<T> results;
        size_t remaining;
        Promise<std::vector<T>> promise;
    }>();

    state->results.resize(futures.size());
    state->remaining = futures.size();
    state->promise = std::move(promise);

    for (size_t i = 0; i < futures.size(); ++i) {
        futures[i].then([state, i](T value) {
            std::lock_guard<std::mutex> lock(state->mutex);
            state->results[i] = std::move(value);

            if (--state->remaining == 0) {
                state->promise.set_value(std::move(state->results));
            }
        });
    }

    return result_future;
}

// 사용법
std::vector<Future<int>> futures;
for (int i = 0; i < 5; ++i) {
    Promise<int> p;
    futures.push_back(p.get_future());
    // p를 이행하는 비동기 작업 실행
}

auto all = when_all(std::move(futures));
std::vector<int> results = all.get();  // 모두 완료될 때까지 대기
```

### Any Of (첫 번째 대기)

```cpp
template<typename T>
Future<T> when_any(std::vector<Future<T>> futures) {
    Promise<T> promise;
    auto result_future = promise.get_future();

    auto state = std::make_shared<struct {
        std::mutex mutex;
        bool fulfilled = false;
        Promise<T> promise;
    }>();

    state->promise = std::move(promise);

    for (auto& future : futures) {
        future.then([state](T value) {
            std::lock_guard<std::mutex> lock(state->mutex);

            if (!state->fulfilled) {
                state->fulfilled = true;
                state->promise.set_value(std::move(value));
            }
        });
    }

    return result_future;
}
```

### Race (가장 먼저 완료되는 것)

```cpp
template<typename T>
Future<T> race(Future<T> f1, Future<T> f2) {
    Promise<T> promise;
    auto future = promise.get_future();

    auto shared_promise = std::make_shared<std::optional<Promise<T>>>(
        std::move(promise));

    auto completion_handler = [shared_promise](T value) {
        auto promise_opt = std::atomic_exchange(shared_promise,
                                                std::optional<Promise<T>>{});
        if (promise_opt) {
            promise_opt->set_value(std::move(value));
        }
    };

    f1.then(completion_handler);
    f2.then(completion_handler);

    return future;
}
```

## 에러 처리

### 예외 전파

```cpp
Promise<int> promise;
auto future = promise.get_future();

std::thread t([promise = std::move(promise)]() mutable {
    try {
        // 시뮬레이션된 에러
        throw std::runtime_error("Computation failed!");
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
});

try {
    int result = future.get();  // 예외를 다시 던짐
} catch (const std::exception& e) {
    std::cout << "Caught: " << e.what() << "\n";
}

t.join();
```

### 에러 복구

```cpp
template<typename T>
class Future {
public:
    // ... 이전 메서드들 ...

    // 에러를 잡고 복구
    template<typename F>
    Future<T> catch_error(F&& handler) {
        Promise<T> promise;
        auto future = promise.get_future();

        then([promise = std::move(promise)](T value) mutable {
            promise.set_value(std::move(value));
        });

        // 에러 핸들러 등록
        // 구현은 SharedState 설계에 따라 다름

        return future;
    }
};

// 사용법
auto result = risky_operation()
    .catch_error([](std::exception_ptr ex) {
        try {
            std::rethrow_exception(ex);
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << ", using default\n";
            return 0;  // 기본값
        }
    });
```

## 완전한 예제: 비동기 웹 크롤러

```cpp
#include <string>
#include <vector>
#include <set>

// 시뮬레이션된 HTTP fetch
Future<std::string> fetch_url(const std::string& url) {
    Promise<std::string> promise;
    auto future = promise.get_future();

    std::thread([url, promise = std::move(promise)]() mutable {
        // 네트워크 지연 시뮬레이션
        std::this_thread::sleep_for(std::chrono::milliseconds(500));

        // 응답 시뮬레이션
        std::string html = "<html>Content from " + url + "</html>";
        promise.set_value(html);
    }).detach();

    return future;
}

// HTML에서 링크 추출
std::vector<std::string> extract_links(const std::string& html) {
    // 간소화된 링크 추출
    return {"http://example.com/page1", "http://example.com/page2"};
}

// 단일 페이지 크롤링
Future<std::vector<std::string>> crawl_page(const std::string& url) {
    return fetch_url(url).then([](const std::string& html) {
        std::cout << "Fetched page, extracting links...\n";
        return extract_links(html);
    });
}

// 여러 페이지를 병렬로 크롤링
Future<std::set<std::string>> crawl_pages(const std::vector<std::string>& urls) {
    std::vector<Future<std::vector<std::string>>> futures;

    for (const auto& url : urls) {
        futures.push_back(crawl_page(url));
    }

    return when_all(std::move(futures)).then([](auto results) {
        std::set<std::string> all_links;
        for (const auto& links : results) {
            all_links.insert(links.begin(), links.end());
        }
        return all_links;
    });
}

int main() {
    std::vector<std::string> seed_urls = {
        "http://example.com",
        "http://example.org"
    };

    auto future = crawl_pages(seed_urls);

    std::set<std::string> all_links = future.get();

    std::cout << "Found " << all_links.size() << " unique links:\n";
    for (const auto& link : all_links) {
        std::cout << "  - " << link << "\n";
    }

    return 0;
}
```

## std::async 실행 정책

```cpp
#include <future>

// async로 실행 (새 스레드 보장)
auto f1 = std::async(std::launch::async, compute);

// deferred로 실행 (지연 평가, 새 스레드 없음)
auto f2 = std::async(std::launch::deferred, compute);
// compute()는 f2.get() 호출 시 실행됨

// 기본 정책으로 실행 (구현체가 결정)
auto f3 = std::async(compute);  // async 또는 deferred일 수 있음

// 예제: 지연 평가
auto lazy = std::async(std::launch::deferred, []{
    std::cout << "Computing...\n";
    return 42;
});

std::cout << "Before get\n";
int result = lazy.get();  // 여기서 "Computing..."이 출력됨
std::cout << "After get: " << result << "\n";
```

## std::packaged_task

```cpp
#include <future>
#include <queue>

// packaged_task는 callable을 래핑하고 future를 제공
std::packaged_task<int(int, int)> task([](int a, int b) {
    return a + b;
});

auto future = task.get_future();

// 태스크 실행 (다른 스레드에서도 가능)
task(3, 4);

std::cout << "Result: " << future.get() << "\n";  // 7

// 일반적인 사용: future를 사용하는 태스크 큐
std::queue<std::packaged_task<void()>> task_queue;

void enqueue_task(std::function<void()> func) {
    std::packaged_task<void()> task(func);
    auto future = task.get_future();
    task_queue.push(std::move(task));
    return future;
}
```

## std::shared_future

```cpp
#include <future>

// 일반 future: 단일 소비자
std::future<int> future = std::async([] { return 42; });
int value = future.get();  // OK
// int value2 = future.get();  // 에러: 두 번 get 불가

// shared_future: 다중 소비자
std::promise<int> promise;
std::shared_future<int> shared = promise.get_future().share();

std::thread t1([shared] {
    std::cout << "T1: " << shared.get() << "\n";
});

std::thread t2([shared] {
    std::cout << "T2: " << shared.get() << "\n";
});

promise.set_value(42);

t1.join();
t2.join();
```

## 성능 고려사항

### Future의 오버헤드

```cpp
// Future 오버헤드 요소:
// - 공유 상태를 위한 힙 할당
// - 동기화 (mutex, condition variable)
// - Continuation을 위한 타입 소거
//
// 매우 세밀한 태스크의 경우 오버헤드가 지배적일 수 있음
// 벤치마크: Future vs 직접 호출
auto start = std::chrono::high_resolution_clock::now();

std::future<int> f = std::async(std::launch::async, []{
    return 1 + 1;  // 사소한 연산
});
int result = f.get();

auto end = std::chrono::high_resolution_clock::now();
// 사소한 태스크에서 Future 오버헤드는 1000배 이상일 수 있음
```

### 최적화: 인라인 Ready Future

```cpp
template<typename T>
class InlineFuture {
    // 값이 즉시 사용 가능하면 할당을 피함
    std::variant<T, std::shared_ptr<SharedState>> value_;

    static InlineFuture make_ready(T value) {
        InlineFuture f;
        f.value_ = std::move(value);
        return f;
    }
};
```

## 흔한 함정

### 1. Get/Wait 호출을 잊는 경우

```cpp
// 잘못된 예: 값을 가져오지 않고 Future가 소멸됨
{
    auto f = std::async(std::launch::async, expensive_task);
}  // 여기서 블로킹! 소멸자가 대기함

// 올바른 예: 명시적으로 결과를 가져옴
auto f = std::async(std::launch::async, expensive_task);
auto result = f.get();
```

### 2. std::future에서 여러 번 Get 호출

```cpp
// 잘못된 예: 한 번만 get 가능
std::future<int> f = std::async([] { return 42; });
int v1 = f.get();  // OK
int v2 = f.get();  // 예외 발생!

// 올바른 예: 다중 소비자에는 shared_future 사용
auto sf = f.share();
int v1 = sf.get();  // OK
int v2 = sf.get();  // OK
```

### 3. 예외 안전성

```cpp
// 잘못된 예: 예외를 잡지 않음
std::thread t([promise = std::move(promise)]() mutable {
    promise.set_value(risky_operation());  // 던질 수 있음!
});

// 올바른 예: 항상 예외를 잡음
std::thread t([promise = std::move(promise)]() mutable {
    try {
        promise.set_value(risky_operation());
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
});
```

## 실제 응용 사례

### 1. 비동기 I/O 연산
```cpp
auto file_future = read_file_async("data.txt");
auto network_future = fetch_url_async("http://api.example.com");

auto results = when_all({file_future, network_future}).get();
```

### 2. 데이터베이스 쿼리
```cpp
auto query1 = db.execute_async("SELECT * FROM users");
auto query2 = db.execute_async("SELECT * FROM orders");

auto [users, orders] = when_all(query1, query2).get();
```

### 3. 병렬 알고리즘
```cpp
auto sort_future = parallel_sort_async(data);
auto filter_future = parallel_filter_async(data);

auto sorted = sort_future.get();
auto filtered = filter_future.get();
```

### 4. 마이크로서비스 통신
```cpp
auto auth_response = auth_service.validate_async(token);
auto user_data = user_service.get_async(user_id);

auto authorized_user = when_all(auth_response, user_data)
    .then([](auto results) {
        auto [auth, user] = results;
        if (auth.valid) return user;
        throw UnauthorizedException();
    });
```

## 다른 패턴과의 비교

### Future vs Callback
```cpp
// 콜백 방식
fetch_url(url, [](Result result) {
    process(result, [](Data data) {
        save(data, [](bool success) {
            // 콜백 지옥!
        });
    });
});

// Future 방식
fetch_url_async(url)
    .then(process)
    .then(save)
    .then([](bool success) {
        // 깔끔하고 합성 가능!
    });
```

## 장단점

### 장점
- 비동기 연산의 깔끔한 합성
- 비동기 경계를 넘는 예외 전파
- 타입 안전한 비동기 프로그래밍
- 표준 라이브러리 지원
- 콜백 지옥 회피
- 취소 지원 (확장 시)

### 단점
- 사소한 연산에 대한 오버헤드
- C++ 표준 라이브러리의 한계 (C++20/23까지 continuation 미지원)
- 공유 상태를 위한 메모리 할당
- std::future 취소 불가
- 복잡한 합성을 위한 학습 곡선

## 모범 사례

1. **비동기 연산에 future 사용**:
   - I/O 연산
   - 네트워크 요청
   - 장시간 실행되는 연산

2. **Continuation으로 체이닝**:
   ```cpp
   result = async_op1()
       .then(async_op2)
       .then(async_op3);
   ```

3. **에러를 우아하게 처리**:
   ```cpp
   result.then(success_handler)
         .catch_error(error_handler);
   ```

4. **브로드캐스트에 shared_future 사용**:
   ```cpp
   std::shared_future<Config> config = load_config().share();
   // 여러 스레드가 config에 접근 가능
   ```

5. **예외 처리를 잊지 말 것**:
   항상 promise.set_value를 try-catch로 감싸기

6. **오버헤드 고려**:
   매우 세밀한 태스크의 경우 직접 호출이 더 빠를 수 있음

## 요약

Future/Promise 패턴은 다음에 필수적입니다:
- 비동기 프로그래밍
- 비동기 연산의 합성
- 비동기 경계를 넘는 깔끔한 에러 처리
- 모던 리액티브 프로그래밍

이 패턴은 미래에 사용 가능해질 값을 표현하고 작업하는 표준적인 방법을 제공하여, 비동기 코드를 더 유지보수하기 쉽고 에러가 적게 만들어줍니다.
