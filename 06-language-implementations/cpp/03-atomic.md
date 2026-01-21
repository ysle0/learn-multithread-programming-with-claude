# Atomic Operations in C++

Atomic operations provide lock-free synchronization for simple data types. They're essential for high-performance concurrent programming and understanding lock-free algorithms.

## Table of Contents
- [Basic Concepts](#basic-concepts)
- [std::atomic Types](#stdatomic-types)
- [Memory Ordering](#memory-ordering)
- [Atomic Operations](#atomic-operations)
- [Compare-and-Swap](#compare-and-swap)
- [Comparison with Other Languages](#comparison-with-other-languages)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)

## Basic Concepts

### What are Atomics?

Atomic operations are indivisible operations that complete without interference from other threads:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

// Non-atomic - race condition!
int counter = 0;

void bad_increment() {
    for (int i = 0; i < 100000; ++i) {
        ++counter;  // Read-modify-write - NOT atomic!
    }
}

// Atomic - safe without locks
std::atomic<int> atomic_counter{0};

void good_increment() {
    for (int i = 0; i < 100000; ++i) {
        ++atomic_counter;  // Atomic operation
    }
}

int main() {
    std::thread t1(good_increment);
    std::thread t2(good_increment);
    t1.join();
    t2.join();
    std::cout << "Result: " << atomic_counter << "\n";  // Always 200000
    return 0;
}
```

### Why Use Atomics?

1. **Lock-Free**: No mutex overhead
2. **Fast**: Hardware-supported operations
3. **Simple**: For basic synchronization
4. **Foundation**: Building block for complex lock-free structures

## std::atomic Types

### Basic Atomic Types

```cpp
#include <atomic>
#include <iostream>

int main() {
    // Integer types
    std::atomic<int> atomic_int{0};
    std::atomic<long> atomic_long{0};
    std::atomic<unsigned> atomic_uint{0};

    // Boolean
    std::atomic<bool> atomic_flag{false};

    // Pointer
    int value = 42;
    std::atomic<int*> atomic_ptr{&value};

    // User-defined types (must be trivially copyable)
    struct Point {
        int x, y;
    };
    std::atomic<Point> atomic_point{{0, 0}};

    return 0;
}
```

### Atomic Type Aliases

```cpp
#include <atomic>

int main() {
    // Convenient type aliases
    std::atomic_int ai{0};           // Same as std::atomic<int>
    std::atomic_long al{0};          // Same as std::atomic<long>
    std::atomic_bool ab{false};      // Same as std::atomic<bool>

    // Fixed-width types
    std::atomic_int32_t ai32{0};
    std::atomic_int64_t ai64{0};

    return 0;
}
```

### std::atomic_flag

The only guaranteed lock-free atomic:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Spin wait
        }
    }

    void unlock() {
        flag.clear(std::memory_order_release);
    }
};

Spinlock spinlock;
int counter = 0;

void increment() {
    for (int i = 0; i < 10000; ++i) {
        spinlock.lock();
        ++counter;
        spinlock.unlock();
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Counter: " << counter << "\n";
    return 0;
}
```

## Memory Ordering

### Memory Order Options

C++ provides fine-grained control over memory synchronization:

```cpp
namespace std {
    enum memory_order {
        memory_order_relaxed,   // No synchronization
        memory_order_consume,   // Data dependency (rarely used)
        memory_order_acquire,   // Acquire barrier
        memory_order_release,   // Release barrier
        memory_order_acq_rel,   // Both acquire and release
        memory_order_seq_cst    // Sequential consistency (default)
    };
}
```

### Sequential Consistency (Default)

Strongest ordering - operations appear in same order to all threads:

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> x{0}, y{0};

void write_x() {
    x.store(1, std::memory_order_seq_cst);  // Default
}

void write_y() {
    y.store(1, std::memory_order_seq_cst);
}

void read_values() {
    int r1 = y.load(std::memory_order_seq_cst);
    int r2 = x.load(std::memory_order_seq_cst);
    // If r1 == 1, then r2 must be 1 (total order guaranteed)
}
```

### Relaxed Ordering

No synchronization, only atomicity:

```cpp
#include <atomic>
#include <thread>

std::atomic<int> counter{0};

void increment_relaxed() {
    for (int i = 0; i < 10000; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(increment_relaxed);
    std::thread t2(increment_relaxed);
    t1.join();
    t2.join();
    // Counter is correct, but no ordering guarantees
    return 0;
}
```

### Acquire-Release Ordering

Most common for synchronization:

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <cassert>

std::atomic<bool> ready{false};
int data = 0;

void producer() {
    data = 42;                                    // 1
    ready.store(true, std::memory_order_release); // 2
    // All writes before release are visible after acquire
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {  // 3
        // Wait
    }
    assert(data == 42);  // Guaranteed to see data = 42
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);
    t1.join();
    t2.join();
    return 0;
}
```

### Memory Ordering Comparison

```cpp
#include <atomic>
#include <thread>

std::atomic<int> value{0};

void examples() {
    // Sequential consistency - strongest, slowest
    value.store(1, std::memory_order_seq_cst);
    int v1 = value.load(std::memory_order_seq_cst);

    // Acquire-release - balanced
    value.store(2, std::memory_order_release);
    int v2 = value.load(std::memory_order_acquire);

    // Relaxed - weakest, fastest
    value.store(3, std::memory_order_relaxed);
    int v3 = value.load(std::memory_order_relaxed);
}
```

## Atomic Operations

### Load and Store

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> value{42};

    // Load
    int v1 = value.load();
    int v2 = value;  // Implicit load

    // Store
    value.store(100);
    value = 200;  // Implicit store

    std::cout << "Value: " << value.load() << "\n";
    return 0;
}
```

### Fetch-and-Add/Sub

```cpp
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

std::atomic<int> counter{0};

void increment_100k() {
    for (int i = 0; i < 100000; ++i) {
        counter.fetch_add(1);  // Returns old value
        // Equivalent to: counter += 1 or ++counter
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(increment_100k);
    }

    for (auto& t : threads) {
        t.join();
    }

    std::cout << "Counter: " << counter << "\n";  // 400000
    return 0;
}
```

### Fetch-and-Or/And/Xor

```cpp
#include <atomic>
#include <iostream>

int main() {
    std::atomic<unsigned int> flags{0};

    // Set bits atomically
    flags.fetch_or(0b0001);   // Set bit 0
    flags.fetch_or(0b0010);   // Set bit 1

    // Clear bits atomically
    flags.fetch_and(~0b0001); // Clear bit 0

    // Toggle bits atomically
    flags.fetch_xor(0b0010);  // Toggle bit 1

    std::cout << "Flags: " << flags << "\n";
    return 0;
}
```

### Exchange

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> value{0};

void swap_values() {
    int old_value = value.exchange(42);  // Set to 42, return old value
    std::cout << "Old value: " << old_value << "\n";
}

int main() {
    std::thread t1(swap_values);
    std::thread t2(swap_values);
    t1.join();
    t2.join();
    std::cout << "Final value: " << value << "\n";
    return 0;
}
```

## Compare-and-Swap

### compare_exchange_weak

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> value{0};

void cas_increment() {
    int expected = value.load();
    while (!value.compare_exchange_weak(expected, expected + 1)) {
        // expected updated with current value on failure
        // Retry
    }
}

int main() {
    std::thread t1(cas_increment);
    std::thread t2(cas_increment);
    t1.join();
    t2.join();
    std::cout << "Value: " << value << "\n";  // 2
    return 0;
}
```

### compare_exchange_strong

```cpp
#include <atomic>
#include <iostream>

struct Node {
    int value;
    Node* next;
};

class LockFreeStack {
    std::atomic<Node*> head{nullptr};

public:
    void push(int value) {
        Node* new_node = new Node{value, nullptr};
        new_node->next = head.load();

        // Keep trying until we successfully update head
        while (!head.compare_exchange_strong(new_node->next, new_node)) {
            // new_node->next updated with current head on failure
        }
    }

    bool pop(int& value) {
        Node* old_head = head.load();
        while (old_head && !head.compare_exchange_strong(old_head, old_head->next)) {
            // old_head updated with current head on failure
        }

        if (old_head) {
            value = old_head->value;
            delete old_head;
            return true;
        }
        return false;
    }
};

int main() {
    LockFreeStack stack;
    stack.push(1);
    stack.push(2);
    stack.push(3);

    int value;
    while (stack.pop(value)) {
        std::cout << value << " ";
    }
    std::cout << "\n";
    return 0;
}
```

### Weak vs. Strong

```cpp
#include <atomic>

std::atomic<int> value{0};

void example_weak() {
    int expected = 0;
    // May fail spuriously (on some architectures)
    // Use in loop
    while (!value.compare_exchange_weak(expected, 1)) {
        // Retry
    }
}

void example_strong() {
    int expected = 0;
    // Never fails spuriously
    // Can use without loop (if you don't retry on failure)
    if (value.compare_exchange_strong(expected, 1)) {
        // Success
    } else {
        // Actual failure (expected != value)
    }
}
```

## Comparison with Other Languages

### C++ vs. C#
```cpp
// C++
std::atomic<int> counter{0};
counter.fetch_add(1);

// C# equivalent:
// int counter = 0;
// Interlocked.Increment(ref counter);
```

### C++ vs. Go
```cpp
// C++
std::atomic<int> counter{0};
counter.store(42);

// Go equivalent:
// var counter int32
// atomic.StoreInt32(&counter, 42)
```

### C++ vs. JavaScript
```cpp
// C++
std::atomic<int> counter{0};
counter.fetch_add(1);

// JavaScript (SharedArrayBuffer + Atomics):
// const buffer = new SharedArrayBuffer(4);
// const view = new Int32Array(buffer);
// Atomics.add(view, 0, 1);
```

## Best Practices

### 1. Use Atomics for Simple Synchronization

```cpp
// GOOD: Simple counter
std::atomic<int> counter{0};
++counter;

// OVERKILL: Don't use mutex for simple counter
std::mutex mtx;
int counter = 0;
{
    std::lock_guard lock(mtx);
    ++counter;
}
```

### 2. Prefer Sequential Consistency Initially

```cpp
// GOOD: Start with seq_cst (default)
std::atomic<int> value{0};
value.store(42);  // memory_order_seq_cst implied

// ADVANCED: Optimize later if needed
value.store(42, std::memory_order_release);
```

### 3. Use Acquire-Release for Synchronization

```cpp
// Producer-consumer pattern
std::atomic<bool> ready{false};
int data;

void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer() {
    while (!ready.load(std::memory_order_acquire));
    // data is visible here
}
```

### 4. Use Relaxed for Counters (When Order Doesn't Matter)

```cpp
// GOOD: Relaxed for simple counters
std::atomic<long> request_count{0};

void handle_request() {
    // Just counting, order doesn't matter
    request_count.fetch_add(1, std::memory_order_relaxed);
}
```

### 5. Check if Type is Lock-Free

```cpp
#include <atomic>
#include <iostream>

struct LargeStruct {
    long data[100];
};

int main() {
    std::atomic<int> atomic_int;
    std::atomic<LargeStruct> atomic_large;

    std::cout << "int is lock-free: "
              << atomic_int.is_lock_free() << "\n";
    std::cout << "LargeStruct is lock-free: "
              << atomic_large.is_lock_free() << "\n";

    // Or at compile time (C++17)
    static_assert(std::atomic<int>::is_always_lock_free);

    return 0;
}
```

## Common Pitfalls

### 1. Assuming All Atomics are Lock-Free

```cpp
// BAD: Might not be lock-free!
struct BigStruct {
    long data[1000];
};
std::atomic<BigStruct> big_atomic;  // Probably uses mutex internally!

// GOOD: Check first
if (!big_atomic.is_lock_free()) {
    std::cout << "Warning: Not lock-free!\n";
}
```

### 2. Mixing Atomic and Non-Atomic Access

```cpp
// BAD: Data race!
std::atomic<int> value{0};

void thread1() {
    value.store(42);  // Atomic
}

void thread2() {
    int* ptr = reinterpret_cast<int*>(&value);
    *ptr = 100;  // Non-atomic - UNDEFINED BEHAVIOR!
}
```

### 3. Forgetting Memory Ordering

```cpp
// BAD: Relaxed may not provide needed synchronization
std::atomic<bool> ready{false};
int data;

void producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // TOO WEAK!
}

void consumer() {
    while (!ready.load(std::memory_order_relaxed));
    // data may not be visible!
}

// GOOD: Use acquire-release
void producer_fixed() {
    data = 42;
    ready.store(true, std::memory_order_release);
}

void consumer_fixed() {
    while (!ready.load(std::memory_order_acquire));
    // data is guaranteed visible
}
```

### 4. ABA Problem

```cpp
// BAD: ABA problem
class Stack {
    std::atomic<Node*> head;

    void pop() {
        Node* old_head = head.load();
        // Thread 1 paused here
        // Thread 2: pop A, pop B, push A (same address!)
        // Thread 1 resumes:
        head.compare_exchange_strong(old_head, old_head->next);
        // Success, but B was lost!
    }
};

// SOLUTION: Use tagged pointers or hazard pointers
```

### 5. False Sharing

```cpp
// BAD: False sharing - atomics on same cache line
struct Counters {
    std::atomic<int> counter1{0};
    std::atomic<int> counter2{0};
};

// GOOD: Separate cache lines
struct Counters {
    alignas(64) std::atomic<int> counter1{0};
    alignas(64) std::atomic<int> counter2{0};
};
```

## Performance Considerations

### Operation Costs

```cpp
// Relative costs (very approximate):
// Relaxed atomic:  1x
// Acquire-release: 1-2x
// Seq_cst:        2-10x
// Mutex lock:     25x (uncontended), 1000x+ (contended)
```

### When to Use What

```cpp
// Simple counter: Relaxed
std::atomic<long> stats{0};
stats.fetch_add(1, std::memory_order_relaxed);

// Flag with data dependency: Acquire-release
std::atomic<bool> ready{false};
int data;
ready.store(true, std::memory_order_release);

// Multiple atomic variables: Sequential consistency
std::atomic<int> x{0}, y{0};
x.store(1);  // seq_cst ensures total order
y.store(1);

// Complex data structure: Use mutex
std::mutex mtx;
ComplexStructure data;
```

## Complete Example: Lock-Free Queue

```cpp
#include <atomic>
#include <memory>
#include <iostream>

template<typename T>
class LockFreeQueue {
    struct Node {
        std::shared_ptr<T> data;
        std::atomic<Node*> next;
        Node() : next(nullptr) {}
    };

    std::atomic<Node*> head;
    std::atomic<Node*> tail;

public:
    LockFreeQueue() {
        Node* dummy = new Node();
        head.store(dummy);
        tail.store(dummy);
    }

    ~LockFreeQueue() {
        while (Node* old_head = head.load()) {
            head.store(old_head->next);
            delete old_head;
        }
    }

    void push(T value) {
        auto data = std::make_shared<T>(std::move(value));
        Node* new_node = new Node();
        Node* old_tail = tail.load();

        while (true) {
            Node* null_ptr = nullptr;
            if (old_tail->next.compare_exchange_strong(null_ptr, new_node)) {
                old_tail->data = data;
                tail.compare_exchange_strong(old_tail, new_node);
                return;
            } else {
                tail.compare_exchange_strong(old_tail, old_tail->next.load());
            }
        }
    }

    std::shared_ptr<T> pop() {
        Node* old_head = head.load();
        while (old_head != tail.load()) {
            if (head.compare_exchange_strong(old_head, old_head->next)) {
                std::shared_ptr<T> result = old_head->next.load()->data;
                delete old_head;
                return result;
            }
        }
        return nullptr;
    }
};

int main() {
    LockFreeQueue<int> queue;

    queue.push(1);
    queue.push(2);
    queue.push(3);

    while (auto value = queue.pop()) {
        std::cout << *value << " ";
    }
    std::cout << "\n";

    return 0;
}
```

## Further Reading

- [C++ Reference: std::atomic](https://en.cppreference.com/w/cpp/atomic/atomic)
- [C++ Memory Model](https://en.cppreference.com/w/cpp/atomic/memory_order)
- "C++ Concurrency in Action" by Anthony Williams
- [Lock-Free Programming](../../05-advanced-patterns/)

## Navigation

- [Back to C++ Overview](./README.md)
- Previous: [Mutex and Lock Guard](./02-mutex-lock-guard.md)
- Next: [Condition Variables](./04-condition-variable.md)
