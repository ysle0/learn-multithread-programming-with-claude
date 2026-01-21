# std::thread - C++ Threading Basics

`std::thread` is the fundamental building block for concurrent programming in C++. Introduced in C++11, it provides a portable, RAII-compliant wrapper around OS threads.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [Creating Threads](#creating-threads)
- [Thread Lifecycle](#thread-lifecycle)
- [Passing Arguments](#passing-arguments)
- [Thread Management](#thread-management)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What is std::thread?

`std::thread` represents a single thread of execution. Each `std::thread` object:
- Maps to one OS thread (1:1 model)
- Has its own stack (typically ~2MB)
- Executes independently
- Must be either joined or detached before destruction

### Header and Namespace
```cpp
#include <thread>
#include <iostream>

// In namespace std
std::thread my_thread;
```

## Creating Threads

### Method 1: Function Pointer
```cpp
#include <thread>
#include <iostream>

void hello() {
    std::cout << "Hello from thread!\n";
}

int main() {
    std::thread t(hello);
    t.join();  // Wait for thread to finish
    return 0;
}
```

### Method 2: Lambda Function
```cpp
#include <thread>
#include <iostream>

int main() {
    std::thread t([] {
        std::cout << "Hello from lambda!\n";
    });
    t.join();
    return 0;
}
```

### Method 3: Function Object (Functor)
```cpp
#include <thread>
#include <iostream>

class Worker {
public:
    void operator()() const {
        std::cout << "Hello from functor!\n";
    }
};

int main() {
    Worker w;
    std::thread t(w);  // Copies Worker object
    t.join();
    return 0;
}
```

### Method 4: Member Function
```cpp
#include <thread>
#include <iostream>

class Task {
public:
    void run(int n) {
        std::cout << "Task running with n=" << n << "\n";
    }
};

int main() {
    Task task;
    std::thread t(&Task::run, &task, 42);
    t.join();
    return 0;
}
```

## Thread Lifecycle

### States of a Thread

```cpp
#include <thread>
#include <iostream>

int main() {
    // 1. Created (not yet representing a thread)
    std::thread t;
    std::cout << "Joinable: " << t.joinable() << "\n";  // false

    // 2. Running
    t = std::thread([] {
        std::cout << "Working...\n";
    });
    std::cout << "Joinable: " << t.joinable() << "\n";  // true

    // 3. Joined (thread finished, object no longer represents a thread)
    t.join();
    std::cout << "Joinable: " << t.joinable() << "\n";  // false

    return 0;
}
```

### Join vs. Detach

#### join(): Wait for Thread Completion
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void work() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Work completed\n";
}

int main() {
    std::thread t(work);
    std::cout << "Waiting for thread...\n";
    t.join();  // Blocks until thread finishes
    std::cout << "Thread joined\n";
    return 0;
}
```

#### detach(): Fire and Forget
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void background_work() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Background work done\n";
}

int main() {
    std::thread t(background_work);
    t.detach();  // Thread continues independently

    std::cout << "Main thread continuing...\n";
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 0;
}
// Note: Detached threads may not complete if main exits early!
```

### RAII Thread Wrapper
```cpp
#include <thread>
#include <iostream>

class ThreadGuard {
    std::thread& t;
public:
    explicit ThreadGuard(std::thread& t_) : t(t_) {}
    ~ThreadGuard() {
        if (t.joinable()) {
            t.join();
        }
    }
    ThreadGuard(ThreadGuard const&) = delete;
    ThreadGuard& operator=(ThreadGuard const&) = delete;
};

void may_throw() {
    throw std::runtime_error("Error!");
}

int main() {
    std::thread t([] {
        std::cout << "Thread working...\n";
    });
    ThreadGuard guard(t);

    // Even if exception thrown, guard ensures thread is joined
    may_throw();
    return 0;
}
```

### C++20 jthread: RAII Thread
```cpp
#include <thread>
#include <iostream>

void work() {
    std::cout << "Working...\n";
}

int main() {
    std::jthread t(work);  // Automatically joins on destruction
    // No need to explicitly join!
    return 0;
}
```

## Passing Arguments

### By Value
```cpp
#include <thread>
#include <iostream>
#include <string>

void print_string(std::string s) {
    std::cout << s << "\n";
}

int main() {
    std::string message = "Hello";
    std::thread t(print_string, message);  // Copies message
    t.join();
    return 0;
}
```

### By Reference (with std::ref)
```cpp
#include <thread>
#include <iostream>
#include <functional>

void increment(int& n) {
    ++n;
}

int main() {
    int value = 0;
    std::thread t(increment, std::ref(value));  // Pass by reference
    t.join();
    std::cout << "Value: " << value << "\n";  // 1
    return 0;
}
```

### Move Semantics
```cpp
#include <thread>
#include <iostream>
#include <memory>

void process(std::unique_ptr<int> ptr) {
    std::cout << "Processing: " << *ptr << "\n";
}

int main() {
    auto ptr = std::make_unique<int>(42);
    std::thread t(process, std::move(ptr));  // Move ownership to thread
    // ptr is now nullptr
    t.join();
    return 0;
}
```

### Multiple Arguments
```cpp
#include <thread>
#include <iostream>
#include <string>

void print_info(int id, const std::string& name, double value) {
    std::cout << "ID: " << id << ", Name: " << name
              << ", Value: " << value << "\n";
}

int main() {
    std::thread t(print_info, 1, "Alice", 3.14);
    t.join();
    return 0;
}
```

## Thread Management

### Getting Thread ID
```cpp
#include <thread>
#include <iostream>

void print_thread_id() {
    std::cout << "Thread ID: " << std::this_thread::get_id() << "\n";
}

int main() {
    std::thread t1(print_thread_id);
    std::thread t2(print_thread_id);

    std::cout << "Main thread ID: " << std::this_thread::get_id() << "\n";
    std::cout << "t1 ID: " << t1.get_id() << "\n";
    std::cout << "t2 ID: " << t2.get_id() << "\n";

    t1.join();
    t2.join();
    return 0;
}
```

### Hardware Concurrency
```cpp
#include <thread>
#include <iostream>
#include <vector>

int main() {
    unsigned int cores = std::thread::hardware_concurrency();
    std::cout << "Number of cores: " << cores << "\n";

    // Create one thread per core
    std::vector<std::thread> threads;
    for (unsigned int i = 0; i < cores; ++i) {
        threads.emplace_back([i] {
            std::cout << "Thread " << i << " on core\n";
        });
    }

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### Sleep and Yield
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void busy_wait() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Busy " << i << "\n";
        std::this_thread::yield();  // Give up time slice
    }
}

void timed_work() {
    std::cout << "Starting...\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Done after 1 second\n";

    auto wake_time = std::chrono::system_clock::now()
                   + std::chrono::milliseconds(500);
    std::this_thread::sleep_until(wake_time);
    std::cout << "Done after 500ms more\n";
}

int main() {
    std::thread t1(busy_wait);
    std::thread t2(timed_work);
    t1.join();
    t2.join();
    return 0;
}
```

### Moving Threads
```cpp
#include <thread>
#include <iostream>

std::thread create_thread() {
    return std::thread([] {
        std::cout << "Thread from function\n";
    });
}

int main() {
    std::thread t1([] {
        std::cout << "Thread 1\n";
    });

    std::thread t2 = std::move(t1);  // t1 no longer valid
    // t1.join();  // ERROR: t1 doesn't represent a thread

    std::thread t3 = create_thread();  // Move from return value

    t2.join();
    t3.join();
    return 0;
}
```

## Comparison with Other Languages

### C++ vs. C#
```cpp
// C++: Explicit join/detach required
std::thread t(work);
t.join();

// C# equivalent:
// Thread t = new Thread(Work);
// t.Start();
// t.Join();
```

### C++ vs. Go
```cpp
// C++: Heavy OS threads
std::thread t(work);
t.join();

// Go: Lightweight goroutines
// go work()
// (No explicit join needed, use sync.WaitGroup)
```

### C++ vs. JavaScript
```cpp
// C++: True threading
std::thread t(work);
t.join();

// JavaScript: Workers (different paradigm)
// const worker = new Worker('worker.js');
// worker.postMessage('data');
```

## Best Practices

### 1. Always Join or Detach
```cpp
// GOOD: Explicit join
std::thread t(work);
t.join();

// GOOD: Explicit detach
std::thread t(work);
t.detach();

// GOOD: Use jthread (C++20)
std::jthread t(work);  // Automatically joins

// BAD: Neither join nor detach
std::thread t(work);
// Destructor will call std::terminate!
```

### 2. Use RAII for Exception Safety
```cpp
class ScopedThread {
    std::thread t;
public:
    explicit ScopedThread(std::thread t_) : t(std::move(t_)) {
        if (!t.joinable()) {
            throw std::logic_error("No thread");
        }
    }
    ~ScopedThread() { t.join(); }
    ScopedThread(ScopedThread const&) = delete;
};
```

### 3. Prefer Task-Based Over Thread-Based
```cpp
// GOOD: Task-based (easier, handles errors better)
auto future = std::async(std::launch::async, work);
future.get();

// OK: Thread-based (when you need fine control)
std::thread t(work);
t.join();
```

### 4. Limit Number of Threads
```cpp
// BAD: Too many threads
for (int i = 0; i < 10000; ++i) {
    std::thread t(work);
    t.detach();
}

// GOOD: Thread pool with limited size
const unsigned int num_threads = std::thread::hardware_concurrency();
std::vector<std::thread> pool;
pool.reserve(num_threads);
for (unsigned int i = 0; i < num_threads; ++i) {
    pool.emplace_back(worker_function);
}
```

### 5. Be Careful with Thread-Local Storage
```cpp
thread_local int counter = 0;  // Each thread has its own copy

void increment() {
    ++counter;
    std::cout << "Thread " << std::this_thread::get_id()
              << " counter: " << counter << "\n";
}
```

## Common Pitfalls

### 1. Forgetting to Join/Detach
```cpp
// BAD: Will call std::terminate
void bad() {
    std::thread t(work);
}  // Oops!

// GOOD
void good() {
    std::jthread t(work);
}  // Automatically joins
```

### 2. Accessing Destroyed Objects
```cpp
// BAD: Reference to destroyed local variable
void bad() {
    int value = 42;
    std::thread t([&] {
        std::cout << value << "\n";  // Undefined behavior!
    });
    t.detach();
}  // value destroyed, but thread still running

// GOOD: Pass by value or ensure lifetime
void good() {
    int value = 42;
    std::thread t([value] {
        std::cout << value << "\n";
    });
    t.join();
}
```

### 3. Double Join
```cpp
// BAD: Can't join twice
std::thread t(work);
t.join();
t.join();  // Undefined behavior!

// GOOD: Check joinable
if (t.joinable()) {
    t.join();
}
```

### 4. Race Condition on cout
```cpp
// BAD: Interleaved output
std::thread t1([] {
    std::cout << "Thread 1\n";
});
std::thread t2([] {
    std::cout << "Thread 2\n";
});

// GOOD: Use mutex or sync
std::mutex cout_mutex;
std::thread t1([&] {
    std::lock_guard lock(cout_mutex);
    std::cout << "Thread 1\n";
});
```

### 5. Exception in Thread
```cpp
// BAD: Exception terminates program
std::thread t([] {
    throw std::runtime_error("Error!");  // Calls std::terminate!
});
t.join();

// GOOD: Catch and handle
std::thread t([] {
    try {
        throw std::runtime_error("Error!");
    } catch (const std::exception& e) {
        std::cerr << "Exception: " << e.what() << "\n";
    }
});
t.join();
```

## Performance Considerations

### Thread Creation Cost
- **Time**: ~100 microseconds to create a thread
- **Memory**: ~2MB stack space per thread
- **Implication**: Don't create threads for trivial tasks

### Context Switching
- **Cost**: 1-10 microseconds per switch
- **Impact**: More threads = more context switches
- **Rule of Thumb**: Don't create more threads than CPU cores for CPU-bound tasks

### Optimal Thread Count
```cpp
// For CPU-bound tasks
unsigned int optimal = std::thread::hardware_concurrency();

// For I/O-bound tasks (can be higher)
unsigned int optimal = std::thread::hardware_concurrency() * 2;
```

## Complete Example: Parallel Sum
```cpp
#include <thread>
#include <vector>
#include <numeric>
#include <iostream>

void partial_sum(const std::vector<int>& data,
                 size_t start, size_t end,
                 long long& result) {
    result = std::accumulate(data.begin() + start,
                            data.begin() + end, 0LL);
}

int main() {
    const size_t data_size = 1'000'000;
    std::vector<int> data(data_size, 1);

    const unsigned int num_threads = std::thread::hardware_concurrency();
    std::vector<std::thread> threads;
    std::vector<long long> results(num_threads);

    size_t chunk_size = data_size / num_threads;

    // Launch threads
    for (unsigned int i = 0; i < num_threads; ++i) {
        size_t start = i * chunk_size;
        size_t end = (i == num_threads - 1) ? data_size
                                             : (i + 1) * chunk_size;
        threads.emplace_back(partial_sum, std::cref(data),
                           start, end, std::ref(results[i]));
    }

    // Join all threads
    for (auto& t : threads) {
        t.join();
    }

    // Combine results
    long long total = std::accumulate(results.begin(), results.end(), 0LL);
    std::cout << "Total sum: " << total << "\n";

    return 0;
}
```

## Further Reading

- [C++ Reference: std::thread](https://en.cppreference.com/w/cpp/thread/thread)
- [Mutex and Lock Guard](./02-mutex-lock-guard.md)
- [Async and Future](./05-async-future.md)

## Navigation

- [Back to C++ Overview](./README.md)
- Next: [Mutex and Lock Guard](./02-mutex-lock-guard.md)
