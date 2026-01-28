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

## Internal Mechanisms

### pthread/WinAPI Wrapper Structure

`std::thread`는 플랫폼별 스레드 API의 얇은 래퍼입니다:

```cpp
// libstdc++ 내부 구조 (단순화)
class thread {
    typedef __gthread_t native_handle_type;  // pthread_t or HANDLE

    struct _State {
        virtual ~_State() = default;
        virtual void _M_run() = 0;  // 실제 작업 수행
    };

    native_handle_type _M_id;  // 스레드 핸들

public:
    template<typename _Callable, typename... _Args>
    explicit thread(_Callable&& __f, _Args&&... __args) {
        // 1. callable과 인수를 decay_copy로 저장
        // 2. __gthread_create 호출
        // 3. 실패 시 std::system_error 던짐
    }
};
```

### Thread Creation System Call Flow

```
std::thread 생성자
    │
    ▼
_M_start_thread() ─────────────────────────────────────────┐
    │                                                      │
    ▼ (Linux)                                              ▼ (Windows)
pthread_create()                                    CreateThread()
    │                                                      │
    ▼                                                      ▼
clone(CLONE_VM | CLONE_FS |                        NtCreateThreadEx()
      CLONE_FILES | CLONE_SIGHAND |                        │
      CLONE_THREAD | ...)                                  ▼
    │                                              커널 스레드 객체 생성
    ▼                                              스택 할당 (Reserved VM)
do_fork() → copy_process()
    │
    ▼
task_struct 할당
스택 할당 (default 8MB, guard page 포함)
TLS 영역 설정 (FS 레지스터)
```

### Thread Stack Layout (Linux x86-64)

```
High Address
┌─────────────────────────────────────┐ ← Stack Top (pthread_attr_t.stackaddr)
│          Arguments/Env             │
├─────────────────────────────────────┤
│             Red Zone               │ ← 128 bytes (leaf function optimization)
├─────────────────────────────────────┤
│          Stack Frames              │
│    ┌─────────────────────────┐    │
│    │ Return Address         │    │
│    │ Saved RBP              │    │
│    │ Local Variables        │    │
│    │ Spilled Registers      │    │
│    └─────────────────────────┘    │
│              ...                   │
├─────────────────────────────────────┤
│         Guard Page(s)              │ ← PROT_NONE (4KB-64KB)
│   (Stack overflow detection)       │
├─────────────────────────────────────┤
│            TLS Block               │ ← FS:0 기준
│  ┌────────────────────────────┐   │
│  │ Static TLS (.tdata)       │   │ ← 음수 오프셋
│  │ pthread struct            │   │
│  │ DTV (Dynamic Thread Vector)│   │
│  └────────────────────────────┘   │
└─────────────────────────────────────┘ ← Stack Bottom
Low Address
```

### join() Internal Implementation

```cpp
// pthread_join 내부 동작 (glibc)
int pthread_join(pthread_t thread, void **retval) {
    struct pthread *pd = (struct pthread *)thread;

    // 1. 이미 join 되었거나 detach 되었는지 확인
    if (pd->joinid != 0)
        return EINVAL;

    // 2. 스레드 종료 대기 (futex 기반)
    while (pd->tid != 0) {
        // FUTEX_WAIT: pd->tid가 현재 값과 같으면 sleep
        futex(&pd->tid, FUTEX_WAIT, pd->tid, NULL, NULL, 0);
    }

    // 3. 반환값 복사
    if (retval)
        *retval = pd->result;

    // 4. 리소스 정리 (스택, TLS 해제)
    __free_tcb(pd);

    return 0;
}
```

**detach()와의 차이**:
```cpp
// detach는 즉시 리소스 정리 책임을 스레드에게 넘김
int pthread_detach(pthread_t thread) {
    struct pthread *pd = (struct pthread *)thread;

    // atomic하게 joinid 설정
    // 스레드 종료 시 자체적으로 리소스 정리
    pd->joinid = pd;  // self-pointer = detached

    return 0;
}
```

### jthread Stop Token Mechanism (C++20)

```cpp
// std::jthread의 협력적 취소 메커니즘
class jthread {
    std::stop_source _M_stop_source;  // 취소 토큰 소스
    std::thread _M_thread;

public:
    template<typename _Callable, typename... _Args>
    explicit jthread(_Callable&& __f, _Args&&... __args) {
        // stop_token을 첫 번째 인수로 전달 (callable이 지원하면)
        if constexpr (std::is_invocable_v<_Callable, stop_token, _Args...>) {
            _M_thread = std::thread(std::forward<_Callable>(__f),
                                   _M_stop_source.get_token(),
                                   std::forward<_Args>(__args)...);
        } else {
            _M_thread = std::thread(std::forward<_Callable>(__f),
                                   std::forward<_Args>(__args)...);
        }
    }

    ~jthread() {
        if (joinable()) {
            request_stop();  // 취소 요청
            join();          // 종료 대기
        }
    }

    bool request_stop() noexcept {
        return _M_stop_source.request_stop();
    }
};

// stop_source 내부: atomic flag + callback 리스트
struct __stop_state {
    std::atomic<uint32_t> _M_owners{1};    // 참조 카운트
    std::atomic<uint32_t> _M_value{0};     // bit 0: stop requested
    __stop_callback_base* _M_callbacks{};  // 콜백 연결 리스트
    std::mutex _M_mtx;
};
```

### Hardware Concurrency Detection

```cpp
// std::thread::hardware_concurrency() 구현
unsigned int hardware_concurrency() noexcept {
#ifdef _WIN32
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    return si.dwNumberOfProcessors;
#else
    // Linux: /sys/devices/system/cpu/online 파싱 또는
    long result = sysconf(_SC_NPROCESSORS_ONLN);
    return (result > 0) ? result : 0;
#endif
}
```

**주의사항**:
- 하이퍼스레딩 시 논리 코어 수 반환 (물리 코어의 2배)
- 컨테이너/VM에서는 제한된 CPU가 아닌 호스트 CPU 수 반환할 수 있음
- NUMA 시스템에서는 노드별 CPU 친화성 고려 필요

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
