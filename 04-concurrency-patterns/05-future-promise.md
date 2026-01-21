# Future/Promise Pattern

## Overview

The Future/Promise pattern represents a value that will be available in the future. A promise is the writable end where a value is set, and a future is the readable end where the value is retrieved. This pattern is essential for asynchronous programming, allowing you to write non-blocking code that composes cleanly.

## Problem Statement

Asynchronous programming without futures leads to:
- Callback hell (deeply nested callbacks)
- Difficult error handling
- Hard to compose async operations
- Thread synchronization complexity
- No standard way to represent pending results

## Solution Architecture

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

Timeline:
1. Create promise/future pair
2. Pass future to consumer
3. Start async operation with promise
4. Consumer waits on future or registers callback
5. Producer sets value/exception on promise
6. Future becomes ready
7. Consumer retrieves value or exception
```

## Basic Implementation (Using std::future)

### Simple Async Computation

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
    // Launch async task
    std::future<int> future = std::async(std::launch::async,
                                         expensive_computation, 42);

    std::cout << "Computation started, doing other work...\n";

    // Do other work while computation runs
    std::this_thread::sleep_for(std::chrono::seconds(1));

    std::cout << "Getting result...\n";
    int result = future.get();  // Blocks until ready

    std::cout << "Result: " << result << "\n";

    return 0;
}
```

### Promise/Future Pair

```cpp
#include <future>
#include <thread>

void producer(std::promise<int> promise) {
    std::this_thread::sleep_for(std::chrono::seconds(1));

    try {
        int result = 42;  // Compute something
        promise.set_value(result);  // Fulfill promise
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> promise;
    std::future<int> future = promise.get_future();

    std::thread t(producer, std::move(promise));

    std::cout << "Waiting for result...\n";
    int result = future.get();  // Blocks until value is set

    std::cout << "Result: " << result << "\n";

    t.join();
    return 0;
}
```

## Advanced Implementation: Composable Futures

### Custom Future with Continuations

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

    // Blocking get
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

    // Non-blocking check
    bool is_ready() const {
        std::lock_guard<std::mutex> lock(state_->mutex);
        return state_->ready;
    }

    // Wait without retrieving value
    void wait() const {
        std::unique_lock<std::mutex> lock(state_->mutex);
        state_->cv.wait(lock, [this] { return state_->ready; });
    }

    // Wait with timeout
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
            // Already ready, execute immediately
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
            // Not ready, register continuation
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

### Usage Example: Chaining Async Operations

```cpp
Promise<int> promise;
auto future = promise.get_future();

// Chain multiple operations
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

// Fulfill promise in another thread
std::thread t([promise = std::move(promise)]() mutable {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    promise.set_value(5);
});

std::string final_result = result.get();
std::cout << "Final: " << final_result << "\n";
// Output: "20"

t.join();
```

## Parallel Execution Patterns

### All Of (Wait for All)

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

// Usage
std::vector<Future<int>> futures;
for (int i = 0; i < 5; ++i) {
    Promise<int> p;
    futures.push_back(p.get_future());
    // Launch async work that fulfills p
}

auto all = when_all(std::move(futures));
std::vector<int> results = all.get();  // Waits for all
```

### Any Of (Wait for First)

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

### Race (First to Complete)

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

## Error Handling

### Exception Propagation

```cpp
Promise<int> promise;
auto future = promise.get_future();

std::thread t([promise = std::move(promise)]() mutable {
    try {
        // Simulated error
        throw std::runtime_error("Computation failed!");
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
});

try {
    int result = future.get();  // Re-throws exception
} catch (const std::exception& e) {
    std::cout << "Caught: " << e.what() << "\n";
}

t.join();
```

### Error Recovery

```cpp
template<typename T>
class Future {
public:
    // ... previous methods ...

    // Catch errors and recover
    template<typename F>
    Future<T> catch_error(F&& handler) {
        Promise<T> promise;
        auto future = promise.get_future();

        then([promise = std::move(promise)](T value) mutable {
            promise.set_value(std::move(value));
        });

        // Register error handler
        // Implementation depends on your SharedState design

        return future;
    }
};

// Usage
auto result = risky_operation()
    .catch_error([](std::exception_ptr ex) {
        try {
            std::rethrow_exception(ex);
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << ", using default\n";
            return 0;  // Default value
        }
    });
```

## Complete Example: Async Web Crawler

```cpp
#include <string>
#include <vector>
#include <set>

// Simulated HTTP fetch
Future<std::string> fetch_url(const std::string& url) {
    Promise<std::string> promise;
    auto future = promise.get_future();

    std::thread([url, promise = std::move(promise)]() mutable {
        // Simulate network delay
        std::this_thread::sleep_for(std::chrono::milliseconds(500));

        // Simulate response
        std::string html = "<html>Content from " + url + "</html>";
        promise.set_value(html);
    }).detach();

    return future;
}

// Extract links from HTML
std::vector<std::string> extract_links(const std::string& html) {
    // Simplified link extraction
    return {"http://example.com/page1", "http://example.com/page2"};
}

// Crawl a single page
Future<std::vector<std::string>> crawl_page(const std::string& url) {
    return fetch_url(url).then([](const std::string& html) {
        std::cout << "Fetched page, extracting links...\n";
        return extract_links(html);
    });
}

// Crawl multiple pages in parallel
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

## std::async Launch Policies

```cpp
#include <future>

// Launch async (guaranteed new thread)
auto f1 = std::async(std::launch::async, compute);

// Launch deferred (lazy evaluation, no new thread)
auto f2 = std::async(std::launch::deferred, compute);
// compute() runs when f2.get() is called

// Launch with default policy (implementation decides)
auto f3 = std::async(compute);  // May be async or deferred

// Example: Lazy evaluation
auto lazy = std::async(std::launch::deferred, []{
    std::cout << "Computing...\n";
    return 42;
});

std::cout << "Before get\n";
int result = lazy.get();  // "Computing..." printed here
std::cout << "After get: " << result << "\n";
```

## std::packaged_task

```cpp
#include <future>
#include <queue>

// Packaged task wraps callable and provides future
std::packaged_task<int(int, int)> task([](int a, int b) {
    return a + b;
});

auto future = task.get_future();

// Execute task (can be in another thread)
task(3, 4);

std::cout << "Result: " << future.get() << "\n";  // 7

// Common use: Task queue with futures
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

// Regular future: single consumer
std::future<int> future = std::async([] { return 42; });
int value = future.get();  // OK
// int value2 = future.get();  // ERROR: Can't get twice

// Shared future: multiple consumers
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

## Performance Considerations

### Overhead of Futures

```cpp
// Future overhead includes:
// - Heap allocation for shared state
// - Synchronization (mutex, condition variable)
// - Type erasure for continuations

// For very fine-grained tasks, overhead can dominate
// Benchmark: Future vs direct call
auto start = std::chrono::high_resolution_clock::now();

std::future<int> f = std::async(std::launch::async, []{
    return 1 + 1;  // Trivial computation
});
int result = f.get();

auto end = std::chrono::high_resolution_clock::now();
// Future overhead can be 1000x+ for trivial tasks
```

### Optimization: Inline Ready Futures

```cpp
template<typename T>
class InlineFuture {
    // If value is immediately available, avoid allocation
    std::variant<T, std::shared_ptr<SharedState>> value_;

    static InlineFuture make_ready(T value) {
        InlineFuture f;
        f.value_ = std::move(value);
        return f;
    }
};
```

## Common Pitfalls

### 1. Forgetting to Get/Wait

```cpp
// BAD: Future destroyed without getting value
{
    auto f = std::async(std::launch::async, expensive_task);
}  // Blocks here! Destructor waits

// GOOD: Explicitly get result
auto f = std::async(std::launch::async, expensive_task);
auto result = f.get();
```

### 2. Multiple Gets on std::future

```cpp
// BAD: Can only get once
std::future<int> f = std::async([] { return 42; });
int v1 = f.get();  // OK
int v2 = f.get();  // EXCEPTION!

// GOOD: Use shared_future for multiple consumers
auto sf = f.share();
int v1 = sf.get();  // OK
int v2 = sf.get();  // OK
```

### 3. Exception Safety

```cpp
// BAD: Exception not caught
std::thread t([promise = std::move(promise)]() mutable {
    promise.set_value(risky_operation());  // Might throw!
});

// GOOD: Always catch exceptions
std::thread t([promise = std::move(promise)]() mutable {
    try {
        promise.set_value(risky_operation());
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
});
```

## Real-World Applications

### 1. Async I/O Operations
```cpp
auto file_future = read_file_async("data.txt");
auto network_future = fetch_url_async("http://api.example.com");

auto results = when_all({file_future, network_future}).get();
```

### 2. Database Queries
```cpp
auto query1 = db.execute_async("SELECT * FROM users");
auto query2 = db.execute_async("SELECT * FROM orders");

auto [users, orders] = when_all(query1, query2).get();
```

### 3. Parallel Algorithms
```cpp
auto sort_future = parallel_sort_async(data);
auto filter_future = parallel_filter_async(data);

auto sorted = sort_future.get();
auto filtered = filter_future.get();
```

### 4. Microservices Communication
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

## Comparison with Other Patterns

### Future vs Callback
```cpp
// Callback style
fetch_url(url, [](Result result) {
    process(result, [](Data data) {
        save(data, [](bool success) {
            // Callback hell!
        });
    });
});

// Future style
fetch_url_async(url)
    .then(process)
    .then(save)
    .then([](bool success) {
        // Clean and composable!
    });
```

## Pros and Cons

### Pros
- Clean composition of async operations
- Exception propagation through async boundaries
- Type-safe async programming
- Standard library support
- Avoids callback hell
- Cancellation support (with extensions)

### Cons
- Overhead for trivial operations
- Limited in C++ standard library (no continuations until C++20/23)
- Memory allocation for shared state
- Can't cancel std::future
- Learning curve for complex composition

## Best Practices

1. **Use futures for async operations**:
   - I/O operations
   - Network requests
   - Long-running computations

2. **Chain with continuations**:
   ```cpp
   result = async_op1()
       .then(async_op2)
       .then(async_op3);
   ```

3. **Handle errors gracefully**:
   ```cpp
   result.then(success_handler)
         .catch_error(error_handler);
   ```

4. **Use shared_future for broadcast**:
   ```cpp
   std::shared_future<Config> config = load_config().share();
   // Multiple threads can access config
   ```

5. **Don't forget exception handling**:
   Always wrap promise.set_value in try-catch

6. **Consider overhead**:
   For very fine-grained tasks, direct calls may be faster

## Summary

Future/Promise pattern is essential for:
- Asynchronous programming
- Composing async operations
- Clean error handling across async boundaries
- Modern reactive programming

It provides a standard way to represent and work with values that will be available in the future, making async code more maintainable and less error-prone.
