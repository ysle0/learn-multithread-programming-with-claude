# Producer-Consumer Pattern

## Overview

The Producer-Consumer pattern is one of the most fundamental concurrency patterns. It decouples threads that produce data (producers) from threads that consume data (consumers) using a shared buffer or queue. This separation allows producers and consumers to operate at different rates and provides natural load balancing.

## Problem Statement

In many applications:
- Data producers and consumers operate at different speeds
- Tight coupling between producers and consumers is undesirable
- You need buffering to handle bursts in production or consumption
- Multiple producers and/or consumers need to work concurrently

## Solution Architecture

```
┌──────────┐         ┌─────────────────┐         ┌──────────┐
│Producer 1│─┐       │                 │       ┌─│Consumer 1│
└──────────┘ │       │  Bounded Queue  │       │ └──────────┘
             ├──────▶│   (Shared)      │──────▶┤
┌──────────┐ │       │  [][][][][]     │       │ ┌──────────┐
│Producer 2│─┘       │                 │       └─│Consumer 2│
└──────────┘         └─────────────────┘         └──────────┘
   produce()          put()    get()              consume()
```

### Key Components

1. **Producers**: Generate data items and add them to the queue
2. **Consumers**: Remove data items from the queue and process them
3. **Shared Queue**: Thread-safe buffer with bounded capacity
4. **Synchronization**: Condition variables for full/empty states

## Basic Implementation (C++)

### Thread-Safe Bounded Queue

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>
#include <chrono>

template<typename T>
class BoundedQueue {
private:
    std::queue<T> queue_;
    size_t capacity_;
    mutable std::mutex mutex_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
    bool closed_ = false;

public:
    explicit BoundedQueue(size_t capacity) : capacity_(capacity) {}

    // Producer: blocking put
    bool put(T item) {
        std::unique_lock<std::mutex> lock(mutex_);

        // Wait until queue is not full or closed
        not_full_.wait(lock, [this] {
            return queue_.size() < capacity_ || closed_;
        });

        if (closed_) {
            return false;  // Queue is closed, can't add
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();  // Wake up a consumer
        return true;
    }

    // Producer: non-blocking try_put
    bool try_put(T item) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (closed_ || queue_.size() >= capacity_) {
            return false;
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();
        return true;
    }

    // Producer: put with timeout
    template<typename Rep, typename Period>
    bool put_for(T item, const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (!not_full_.wait_for(lock, timeout, [this] {
            return queue_.size() < capacity_ || closed_;
        })) {
            return false;  // Timeout
        }

        if (closed_) {
            return false;
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();
        return true;
    }

    // Consumer: blocking get
    std::optional<T> get() {
        std::unique_lock<std::mutex> lock(mutex_);

        // Wait until queue is not empty or closed
        not_empty_.wait(lock, [this] {
            return !queue_.empty() || closed_;
        });

        if (queue_.empty()) {
            return std::nullopt;  // Queue is closed and empty
        }

        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();  // Wake up a producer
        return item;
    }

    // Consumer: non-blocking try_get
    std::optional<T> try_get() {
        std::unique_lock<std::mutex> lock(mutex_);

        if (queue_.empty()) {
            return std::nullopt;
        }

        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }

    // Consumer: get with timeout
    template<typename Rep, typename Period>
    std::optional<T> get_for(const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (!not_empty_.wait_for(lock, timeout, [this] {
            return !queue_.empty() || closed_;
        })) {
            return std::nullopt;  // Timeout
        }

        if (queue_.empty()) {
            return std::nullopt;  // Closed and empty
        }

        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }

    // Close the queue (no more items can be added)
    void close() {
        std::lock_guard<std::mutex> lock(mutex_);
        closed_ = true;
        not_full_.notify_all();   // Wake all waiting producers
        not_empty_.notify_all();  // Wake all waiting consumers
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.size();
    }

    bool is_closed() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return closed_;
    }
};
```

### Complete Example: Image Processing Pipeline

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <string>
#include <chrono>
#include <random>

// Simulated image data
struct Image {
    int id;
    std::string filename;
    std::vector<uint8_t> data;

    Image(int id, const std::string& name)
        : id(id), filename(name), data(1024 * 1024) {}  // 1MB
};

// Producer: Load images from disk
void image_loader(BoundedQueue<Image>& queue, int count) {
    std::cout << "Loader thread started\n";

    for (int i = 0; i < count; ++i) {
        // Simulate loading image from disk
        std::this_thread::sleep_for(std::chrono::milliseconds(100));

        Image img(i, "image_" + std::to_string(i) + ".jpg");
        std::cout << "Loaded: " << img.filename << "\n";

        if (!queue.put(std::move(img))) {
            std::cout << "Queue closed, loader exiting\n";
            break;
        }
    }

    std::cout << "Loader finished\n";
}

// Consumer: Process images
void image_processor(BoundedQueue<Image>& queue, int worker_id) {
    std::cout << "Processor " << worker_id << " started\n";

    while (true) {
        auto img = queue.get();

        if (!img.has_value()) {
            std::cout << "Processor " << worker_id << " exiting\n";
            break;
        }

        // Simulate image processing
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        std::cout << "Processor " << worker_id << " processed: "
                  << img->filename << "\n";
    }
}

int main() {
    const int NUM_IMAGES = 20;
    const int NUM_PROCESSORS = 3;
    const int QUEUE_CAPACITY = 5;

    BoundedQueue<Image> queue(QUEUE_CAPACITY);

    // Start producer
    std::thread loader(image_loader, std::ref(queue), NUM_IMAGES);

    // Start consumers
    std::vector<std::thread> processors;
    for (int i = 0; i < NUM_PROCESSORS; ++i) {
        processors.emplace_back(image_processor, std::ref(queue), i);
    }

    // Wait for producer to finish
    loader.join();

    // Close queue to signal consumers
    queue.close();

    // Wait for all consumers
    for (auto& p : processors) {
        p.join();
    }

    std::cout << "All processing complete\n";
    return 0;
}
```

## Advanced Variant: Multiple Queues with Priorities

```cpp
#include <array>

template<typename T, size_t NumPriorities = 3>
class PriorityBoundedQueue {
private:
    std::array<std::queue<T>, NumPriorities> queues_;
    size_t capacity_;
    size_t total_size_ = 0;
    mutable std::mutex mutex_;
    std::condition_variable not_full_;
    std::condition_variable not_empty_;
    bool closed_ = false;

public:
    explicit PriorityBoundedQueue(size_t capacity) : capacity_(capacity) {}

    // Put with priority (0 = highest priority)
    bool put(T item, size_t priority = NumPriorities - 1) {
        if (priority >= NumPriorities) {
            priority = NumPriorities - 1;
        }

        std::unique_lock<std::mutex> lock(mutex_);
        not_full_.wait(lock, [this] {
            return total_size_ < capacity_ || closed_;
        });

        if (closed_) {
            return false;
        }

        queues_[priority].push(std::move(item));
        ++total_size_;
        not_empty_.notify_one();
        return true;
    }

    // Get highest priority item available
    std::optional<T> get() {
        std::unique_lock<std::mutex> lock(mutex_);
        not_empty_.wait(lock, [this] {
            return total_size_ > 0 || closed_;
        });

        if (total_size_ == 0) {
            return std::nullopt;
        }

        // Find highest priority non-empty queue
        for (auto& q : queues_) {
            if (!q.empty()) {
                T item = std::move(q.front());
                q.pop();
                --total_size_;
                not_full_.notify_one();
                return item;
            }
        }

        return std::nullopt;
    }

    void close() {
        std::lock_guard<std::mutex> lock(mutex_);
        closed_ = true;
        not_full_.notify_all();
        not_empty_.notify_all();
    }
};
```

## Internal Mechanisms

### Condition Variable 기반 구현의 내부 동작

Bounded Queue의 `put()`과 `get()` 연산이 커널 수준에서 어떻게 동작하는지 살펴봅니다.

#### wait() 내부 동작

```c
// pthread_cond_wait()의 내부 동작 (glibc 간략화)
int __pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex) {
    // 1. futex 값 스냅샷 (대기 전 상태)
    unsigned int seq = cond->__data.__wseq;

    // 2. 대기자 카운트 증가
    atomic_fetch_add(&cond->__data.__nwaiters, 1);

    // 3. Mutex unlock (원자적으로 진행)
    __pthread_mutex_unlock(mutex);

    // 4. futex 대기 (커널 진입)
    // 조건: cond의 sequence가 변경되지 않았다면 sleep
    futex(&cond->__data.__wseq, FUTEX_WAIT, seq, NULL);

    // 5. 깨어남: Mutex 재획득
    __pthread_mutex_lock(mutex);

    // 6. 대기자 카운트 감소
    atomic_fetch_sub(&cond->__data.__nwaiters, 1);

    return 0;
}
```

#### notify_one() 내부 동작

```c
int __pthread_cond_signal(pthread_cond_t *cond) {
    // 대기자가 없으면 아무것도 안 함
    if (atomic_load(&cond->__data.__nwaiters) == 0)
        return 0;

    // sequence 증가
    atomic_fetch_add(&cond->__data.__wseq, 1);

    // futex wake: 하나의 대기자만 깨움
    futex(&cond->__data.__wseq, FUTEX_WAKE, 1);

    return 0;
}
```

### SPSC Ring Buffer의 메모리 순서

```
┌─────────────────────────────────────────────────────────────┐
│              Single Producer Single Consumer                 │
│                                                             │
│  Producer (쓰기 스레드)          Consumer (읽기 스레드)      │
│                                                             │
│  1. buffer[tail] = item         1. item = buffer[head]      │
│     ↓ (release)                    ↑ (acquire)              │
│  2. tail.store(next_tail)       2. head가 tail보다 앞인지   │
│                                    확인                      │
│                                 3. head.store(next_head)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

메모리 순서 분석:

Producer의 store(release):
  - buffer[tail] = item 이 tail 업데이트 전에 완료됨을 보장
  - Consumer가 새 tail을 보면 데이터도 반드시 보임

Consumer의 load(acquire):
  - tail.load() 이후의 buffer 읽기가 재배치되지 않음
  - Producer가 쓴 데이터를 올바르게 읽음
```

#### 캐시 라인 최적화

```cpp
// False Sharing 방지를 위한 패딩
template<typename T, size_t Capacity>
class SPSCQueue {
private:
    // head와 tail을 서로 다른 캐시 라인에 배치
    alignas(64) std::atomic<size_t> head_{0};  // Consumer만 수정
    alignas(64) std::atomic<size_t> tail_{0};  // Producer만 수정

    // 버퍼도 별도 캐시 라인
    alignas(64) std::array<T, Capacity> buffer_;

    /*
     * 메모리 레이아웃:
     *
     * Cache Line 0: [head_][padding.................]
     * Cache Line 1: [tail_][padding.................]
     * Cache Line 2+: [buffer_........................]
     *
     * 이렇게 하면 Producer와 Consumer가 서로의 캐시 라인을
     * 무효화하지 않음 → 성능 향상
     */
};
```

### Bounded Queue의 Backpressure 메커니즘

```
Producer 속도 > Consumer 속도인 경우:

시간 ─────────────────────────────────────────────────────▶

Producer: [produce][produce][produce][BLOCKED........][produce]
                                      ↑
                                      큐가 가득 참
                                      futex_wait

Consumer: [consume].....[consume].....[consume][consume][consume]
                                       ↑
                                       Consumer가 따라잡음
                                       futex_wake → Producer 재개

Backpressure 효과:
  - Producer가 자연스럽게 throttle됨
  - 메모리 사용량 제한 (bounded)
  - 시스템 과부하 방지
```

### Multi-Producer Multi-Consumer (MPMC) 내부 구조

```cpp
// MPMC 큐에서의 슬롯별 시퀀스 번호 기법 (Dmitry Vyukov)
template<typename T, size_t Capacity>
class MPMCQueue {
    struct Slot {
        std::atomic<size_t> sequence;  // 슬롯 상태 추적
        T data;
    };

    alignas(64) Slot buffer_[Capacity];
    alignas(64) std::atomic<size_t> enqueue_pos_{0};
    alignas(64) std::atomic<size_t> dequeue_pos_{0};

    /*
     * 슬롯 sequence의 의미:
     *
     * sequence == pos:      슬롯이 비어있음, enqueue 가능
     * sequence == pos + 1:  슬롯에 데이터 있음, dequeue 가능
     *
     * 초기화: sequence[i] = i
     *
     * Enqueue:
     *   1. pos = enqueue_pos_.fetch_add(1)
     *   2. slot = &buffer_[pos % Capacity]
     *   3. while (slot->sequence.load() != pos) spin; // 빈 슬롯 대기
     *   4. slot->data = item
     *   5. slot->sequence.store(pos + 1)  // 채워짐 표시
     *
     * Dequeue:
     *   1. pos = dequeue_pos_.fetch_add(1)
     *   2. slot = &buffer_[pos % Capacity]
     *   3. while (slot->sequence.load() != pos + 1) spin; // 데이터 대기
     *   4. item = slot->data
     *   5. slot->sequence.store(pos + Capacity)  // 비워짐 표시
     */
};
```

## Lock-Free Implementation (Single Producer, Single Consumer)

```cpp
#include <atomic>
#include <array>

template<typename T, size_t Capacity>
class SPSCQueue {
private:
    struct alignas(64) {  // Cache line alignment
        std::atomic<size_t> head{0};
    };
    struct alignas(64) {
        std::atomic<size_t> tail{0};
    };

    std::array<T, Capacity> buffer_;

public:
    // Producer only
    bool try_push(const T& item) {
        const size_t current_tail = tail.load(std::memory_order_relaxed);
        const size_t next_tail = (current_tail + 1) % Capacity;

        if (next_tail == head.load(std::memory_order_acquire)) {
            return false;  // Queue full
        }

        buffer_[current_tail] = item;
        tail.store(next_tail, std::memory_order_release);
        return true;
    }

    // Consumer only
    bool try_pop(T& item) {
        const size_t current_head = head.load(std::memory_order_relaxed);

        if (current_head == tail.load(std::memory_order_acquire)) {
            return false;  // Queue empty
        }

        item = buffer_[current_head];
        head.store((current_head + 1) % Capacity, std::memory_order_release);
        return true;
    }
};
```

## Performance Considerations

### Queue Capacity Selection

```
Too Small (capacity = 1-2):
  - Frequent blocking
  - Poor throughput
  - Tight coupling

Optimal (capacity = 10-100):
  - Absorbs bursts
  - Good throughput
  - Balanced coupling

Too Large (capacity = 1000+):
  - Memory waste
  - Poor cache locality
  - Delayed backpressure
```

### Benchmarking Code

```cpp
#include <chrono>
#include <numeric>

template<typename QueueType>
void benchmark_throughput(size_t num_items) {
    QueueType queue(100);
    std::atomic<bool> done{false};
    std::atomic<size_t> consumed{0};

    auto start = std::chrono::high_resolution_clock::now();

    // Producer thread
    std::thread producer([&] {
        for (size_t i = 0; i < num_items; ++i) {
            queue.put(i);
        }
        queue.close();
    });

    // Consumer thread
    std::thread consumer([&] {
        while (auto item = queue.get()) {
            consumed.fetch_add(1, std::memory_order_relaxed);
        }
    });

    producer.join();
    consumer.join();

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << "Throughput: " << (num_items * 1000.0 / duration)
              << " items/sec\n";
}
```

## Common Pitfalls

### 1. Forgetting to Close the Queue

```cpp
// BAD: Consumers will wait forever
queue.put(item);
// ... producer finishes but doesn't close queue

// GOOD: Always close when done producing
queue.put(item);
queue.close();  // Signal consumers to stop
```

### 2. Lost Wakeups

```cpp
// BAD: Check condition outside lock
if (queue.empty()) {  // Race: queue might not be empty anymore
    wait_on_condition_variable();
}

// GOOD: Check condition inside wait predicate
lock.wait([&]{ return !queue.empty(); });
```

### 3. Spurious Wakeups Not Handled

```cpp
// BAD: Single check
lock.wait();
auto item = queue.front();  // Might still be empty!

// GOOD: Loop or predicate
lock.wait([&]{ return !queue.empty(); });
auto item = queue.front();
```

## Variants and Extensions

### 1. Work Stealing Queue
Multiple consumers can steal work from each other's local queues.

### 2. Buffered Channel (Go-style)
Combination of buffered and unbuffered communication.

### 3. Disruptor Pattern
Ultra-high-performance ring buffer for low-latency systems.

## Real-World Applications

### 1. Thread Pool Task Queues
```
Task producers → Task queue → Worker threads
```

### 2. Message Brokers
```
Publishers → Message queue → Subscribers
```

### 3. Data Processing Pipelines
```
Data ingest → Processing stages → Output
```

### 4. UI Event Handling
```
User events → Event queue → Event handler thread
```

### 5. Log Aggregation
```
Multiple services → Log queue → Log processor
```

## Pros and Cons

### Pros
- Decouples producers from consumers
- Natural load balancing across consumers
- Absorbs bursts in production/consumption rates
- Simple to understand and implement
- Works well with thread pools

### Cons
- Bounded queues can cause producer blocking
- Unbounded queues can cause memory exhaustion
- Potential for high contention on queue locks
- Doesn't work well for request-response patterns
- Queue tuning can be application-specific

## Best Practices

1. **Choose appropriate queue capacity** based on:
   - Item size
   - Production/consumption rates
   - Memory constraints
   - Latency requirements

2. **Provide timeout variants** for production systems:
   ```cpp
   if (!queue.put_for(item, 5s)) {
       // Handle timeout
   }
   ```

3. **Monitor queue depth** to detect:
   - Consumer starvation
   - Producer overload
   - System imbalances

4. **Use move semantics** to avoid copies:
   ```cpp
   queue.put(std::move(large_object));
   ```

5. **Consider lock-free implementations** for:
   - Single producer, single consumer
   - Ultra-low latency requirements
   - High-frequency trading

## Testing Strategies

### Stress Test
```cpp
// Test with many producers and consumers
const int NUM_PRODUCERS = 10;
const int NUM_CONSUMERS = 10;
const int ITEMS_PER_PRODUCER = 10000;
// Verify all items are produced and consumed exactly once
```

### Fairness Test
```cpp
// Verify no consumer starves
// Verify load is balanced across consumers
```

### Backpressure Test
```cpp
// Slow consumer, fast producer
// Verify producers block appropriately
```

## Summary

The Producer-Consumer pattern is essential for:
- Decoupling system components
- Building scalable data processing systems
- Handling variable workload rates
- Implementing work distribution

Master this pattern as it forms the foundation for many other concurrency patterns and is used extensively in real-world systems.
