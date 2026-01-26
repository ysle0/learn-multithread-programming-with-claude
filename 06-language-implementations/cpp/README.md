# C++ Concurrency

C++ provides low-level, high-performance concurrency primitives starting with C++11. The language follows a "zero-overhead abstraction" philosophy, giving you maximum control while providing safe, modern abstractions.

## Overview

C++ concurrency evolved significantly:
- **C++11**: Introduced threading library, atomics, memory model
- **C++14**: Minor improvements and bug fixes
- **C++17**: Parallel algorithms
- **C++20**: Coroutines, semaphores, barriers, latches, jthread
- **C++23**: Further improvements to synchronization primitives

## Core Components

### 1. [std::thread](./01-std-thread.md)
- Creating and managing OS threads
- Thread lifecycle and joining
- Passing arguments to threads
- Thread IDs and hardware concurrency

### 2. [Mutex and Lock Guard](./02-mutex-lock-guard.md)
- `std::mutex` for mutual exclusion
- RAII lock guards (`std::lock_guard`, `std::unique_lock`)
- Shared mutexes for reader-writer scenarios
- Recursive and timed mutexes

### 3. [Atomic Operations](./03-atomic.md)
- `std::atomic<T>` for lock-free programming
- Memory ordering and synchronization
- Compare-and-swap operations
- Atomic smart pointers (C++20)

### 4. [Condition Variables](./04-condition-variable.md)
- Waiting for conditions to become true
- Producer-consumer patterns
- Spurious wakeups and predicate loops
- Notify one vs. notify all

### 5. [Async and Future](./05-async-future.md)
- Task-based parallelism with `std::async`
- `std::future` and `std::promise`
- Launch policies (async vs. deferred)
- Shared futures and packaged tasks

## 병렬 프로그래밍 도구 (Parallel Programming Tools)

### 6. [C++17/20 병렬 알고리즘](./06-parallel-algorithms.md) 🌟
- 실행 정책 (seq, par, par_unseq)
- 병렬 sort, reduce, transform
- 성능 벤치마크 및 최적화
- 실전 예제: 이미지 처리, 통계 계산

### 7. [C++20/23 동기화 기능](./07-cpp20-synchronization.md) 🆕
- `std::jthread` - 자동 조인 스레드
- `std::stop_token` - 협력적 취소
- `std::counting_semaphore`, `std::binary_semaphore`
- `std::latch`, `std::barrier` - 동기화 지점
- `std::atomic<std::shared_ptr<T>>`

### 8. [Intel TBB](./08-intel-tbb.md) 🚀
- 태스크 기반 병렬화
- parallel_for, parallel_reduce, parallel_scan
- concurrent_vector, concurrent_hash_map
- task_group, parallel_pipeline
- Work-stealing 스케줄러

### 9. [OpenMP](./09-openmp.md) ⚡
- Pragma 기반 병렬화
- parallel for, reduction, sections
- schedule 최적화 (static, dynamic, guided)
- collapse, task, critical
- 가장 간단한 병렬화 방법

## Quick Comparison with Other Languages

| Feature | C++ | Comparison |
|---------|-----|------------|
| **Thread Creation** | `std::thread` | Similar to C# `Thread`, heavier than Go goroutines |
| **Async Pattern** | `std::async` + `std::future` | Less ergonomic than C# `async/await` or JS Promises |
| **Message Passing** | No built-in | Unlike Go channels; use libraries or manual queues |
| **Memory Model** | Well-defined (C++11) | Most explicit and low-level of all four languages |
| **Safety** | Manual, error-prone | Less safe than C#/Go/JS; requires careful programming |

## Key Principles

### 1. RAII (Resource Acquisition Is Initialization)
```cpp
{
    std::lock_guard<std::mutex> lock(mutex);
    // Critical section - lock automatically released when scope ends
}
```

### 2. Zero-Overhead Abstraction
C++ threading primitives compile to efficient machine code with minimal runtime overhead.

### 3. Explicit Memory Ordering
You control exactly how memory operations are synchronized:
```cpp
atomic_var.store(value, std::memory_order_release);
auto val = atomic_var.load(std::memory_order_acquire);
```

## Common Patterns

### Thread-Safe Singleton
```cpp
class Singleton {
    static Singleton& getInstance() {
        static Singleton instance;  // Thread-safe in C++11+
        return instance;
    }
};
```

### Scoped Locking
```cpp
std::mutex m1, m2;
{
    std::scoped_lock lock(m1, m2);  // C++17: locks both, avoids deadlock
    // Critical section
}
```

## Best Practices

1. **Prefer High-Level Abstractions**: Use `std::async` over manual threads when possible
2. **Use RAII**: Always use lock guards, never lock/unlock manually
3. **Avoid Shared State**: Minimize sharing between threads
4. **Const Correctness**: Const data can be safely shared
5. **Use Atomics Carefully**: Understand memory ordering before using relaxed atomics
6. **Document Thread Safety**: Mark which functions are thread-safe
7. **Prefer Value Semantics**: Pass by value with move semantics when possible

## Common Pitfalls

### 1. Forgetting to Join or Detach
```cpp
// BAD: Thread destructor will call std::terminate
void bad_example() {
    std::thread t([] { /* work */ });
}  // Oops! Didn't join or detach

// GOOD: Use jthread (C++20) or ensure join/detach
void good_example() {
    std::jthread t([] { /* work */ });
}  // Automatically joins
```

### 2. Deadlock with Multiple Mutexes
```cpp
// BAD: Can deadlock
mutex1.lock();
mutex2.lock();

// GOOD: Use scoped_lock
std::scoped_lock lock(mutex1, mutex2);
```

### 3. Data Races with Shared Data
```cpp
// BAD: Data race
int counter = 0;
std::thread t1([&] { ++counter; });
std::thread t2([&] { ++counter; });

// GOOD: Use atomic or mutex
std::atomic<int> counter{0};
std::thread t1([&] { ++counter; });
std::thread t2([&] { ++counter; });
```

### 4. Exception Safety
```cpp
// BAD: Lock not released if exception thrown
mutex.lock();
might_throw();  // Lock never released!
mutex.unlock();

// GOOD: Use RAII
{
    std::lock_guard lock(mutex);
    might_throw();  // Lock released even if exception thrown
}
```

### 5. Spurious Wakeups with Condition Variables
```cpp
// BAD: Might wake up when condition isn't met
cv.wait(lock);
process_data();

// GOOD: Use predicate
cv.wait(lock, [] { return data_ready; });
process_data();
```

## Performance Considerations

### Thread Creation Overhead
- Creating a thread: ~100 microseconds
- Context switch: ~1-10 microseconds
- Memory overhead: ~2MB per thread (stack size)

**Implication**: Don't create threads for short-lived tasks; use thread pools.

### Lock Contention
- Uncontended lock: ~25 nanoseconds
- Contended lock: Can be 1000x slower

**Implication**: Minimize time in critical sections.

### False Sharing
```cpp
// BAD: False sharing - counters on same cache line
struct Counters {
    std::atomic<int> counter1;
    std::atomic<int> counter2;
};

// GOOD: Prevent false sharing with alignment
struct Counters {
    alignas(64) std::atomic<int> counter1;
    alignas(64) std::atomic<int> counter2;
};
```

## Modern C++ Features (C++20 and Beyond)

### jthread (Joinable Thread)
```cpp
std::jthread t([] {
    // Work
});  // Automatically joins on destruction
```

### Semaphores
```cpp
std::counting_semaphore<10> sem(3);  // Max count 10, initial count 3
sem.acquire();  // Decrement
sem.release();  // Increment
```

### Barriers and Latches
```cpp
std::barrier sync_point(num_threads);
// Each thread:
sync_point.arrive_and_wait();  // Synchronize all threads
```

### Coroutines (C++20)
```cpp
Task<int> async_computation() {
    co_await some_async_operation();
    co_return 42;
}
```

## 병렬 프로그래밍 도구 선택 가이드

| 도구 | 난이도 | 성능 | 이식성 | 사용 사례 |
|-----|-------|-----|--------|----------|
| **OpenMP** | ⭐ 쉬움 | ⭐⭐⭐⭐ 우수 | ⭐⭐⭐⭐⭐ 최고 | 과학 계산, 간단한 병렬화 |
| **C++17 Parallel Algorithms** | ⭐⭐ 보통 | ⭐⭐⭐⭐ 우수 | ⭐⭐⭐⭐⭐ 최고 | STL 알고리즘 병렬화 |
| **Intel TBB** | ⭐⭐⭐ 중간 | ⭐⭐⭐⭐⭐ 최고 | ⭐⭐⭐⭐ 좋음 | 복잡한 병렬화, 태스크 기반 |
| **C++20 Features** | ⭐⭐ 보통 | ⭐⭐⭐⭐ 우수 | ⭐⭐⭐ 보통 | 현대적인 동기화 (C++20+) |
| **std::thread** | ⭐⭐⭐⭐ 어려움 | ⭐⭐⭐ 보통 | ⭐⭐⭐⭐⭐ 최고 | 저수준 제어 필요 시 |

### 추천 사용 순서
1. **초급자**: OpenMP로 시작 → C++17 Parallel Algorithms
2. **중급자**: Intel TBB → C++20 Features
3. **고급자**: 상황에 맞게 조합 사용

## Recommended Libraries

While C++ standard library is powerful, these libraries can help:

- **Intel TBB**: Thread building blocks for parallel algorithms ⭐ 추천
- **OpenMP**: Compiler directives for easy parallelization ⭐ 추천
- **Boost.Thread**: Extended threading utilities
- **Boost.Asio**: Async I/O and networking
- **folly**: Facebook's C++ library with concurrent data structures
- **libcds**: Lock-free data structures

## Tools and Debugging

### Thread Sanitizer
```bash
g++ -fsanitize=thread -g program.cpp
./a.out
```

### Valgrind (Helgrind)
```bash
valgrind --tool=helgrind ./program
```

### GDB Threading Commands
```
info threads          # List all threads
thread <n>            # Switch to thread n
thread apply all bt   # Backtrace of all threads
```

## Compiler Support

| Feature | GCC | Clang | MSVC |
|---------|-----|-------|------|
| C++11 threads | 4.8+ | 3.3+ | VS2012+ |
| C++17 parallel algorithms | 9+ | Not fully | VS2017+ |
| C++20 jthread | 10+ | 14+ | VS2019 16.9+ |
| C++20 semaphore | 11+ | 11+ | VS2019 16.10+ |
| C++20 coroutines | 10+ | 5+ | VS2019+ |

## Further Reading

- **C++ Concurrency in Action** by Anthony Williams (2nd edition)
- **C++ High Performance** by Björn Andrist and Viktor Sehr
- CppReference: https://en.cppreference.com/w/cpp/thread
- ISO C++ Papers: https://isocpp.org/std/status

## Navigation

- [Back to Language Implementations](../)
- Core Topics:
  - [std::thread](./01-std-thread.md)
  - [Mutex and Lock Guard](./02-mutex-lock-guard.md)
  - [Atomic Operations](./03-atomic.md)
  - [Condition Variables](./04-condition-variable.md)
  - [Async and Future](./05-async-future.md)
- Parallel Programming Tools:
  - [C++17/20 Parallel Algorithms](./06-parallel-algorithms.md)
  - [C++20/23 Synchronization Features](./07-cpp20-synchronization.md)
  - [Intel TBB](./08-intel-tbb.md)
  - [OpenMP](./09-openmp.md)
