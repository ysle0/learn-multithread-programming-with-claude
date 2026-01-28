# Reader-Writer Pattern

## Overview

The Reader-Writer pattern (also known as Shared-Exclusive Lock pattern) optimizes concurrent access to shared data when reads vastly outnumber writes. It allows multiple readers to access data simultaneously while ensuring writers get exclusive access. This pattern is fundamental for building high-performance concurrent data structures.

## Problem Statement

In many systems:
- Read operations far outnumber write operations (90%+ reads)
- Multiple readers can safely access data concurrently
- Writers need exclusive access to maintain consistency
- Simple mutual exclusion (mutex) serializes all access, wasting concurrency potential

## Solution Architecture

```
┌─────────────────────────────────────────────┐
│           Shared Resource                   │
│                                             │
│  Multiple Readers (Concurrent)              │
│  ┌────────┐  ┌────────┐  ┌────────┐       │
│  │Reader 1│  │Reader 2│  │Reader 3│       │
│  └────────┘  └────────┘  └────────┘       │
│                                             │
│           OR (mutually exclusive)           │
│                                             │
│  Single Writer (Exclusive)                  │
│  ┌────────┐                                │
│  │Writer 1│                                │
│  └────────┘                                │
└─────────────────────────────────────────────┘

States:
1. No access
2. One or more readers (no writers)
3. One writer (no readers, no other writers)
```

## Basic Implementation (C++17)

### Using std::shared_mutex

```cpp
#include <shared_mutex>
#include <mutex>
#include <map>
#include <string>

template<typename Key, typename Value>
class ConcurrentMap {
private:
    std::map<Key, Value> data_;
    mutable std::shared_mutex mutex_;

public:
    // Reader: Acquires shared lock
    std::optional<Value> get(const Key& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);

        auto it = data_.find(key);
        if (it != data_.end()) {
            return it->second;
        }
        return std::nullopt;
    }

    // Reader: Check if key exists
    bool contains(const Key& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        return data_.find(key) != data_.end();
    }

    // Reader: Get size
    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        return data_.size();
    }

    // Writer: Acquires exclusive lock
    void put(const Key& key, const Value& value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_[key] = value;
    }

    // Writer: Remove entry
    bool remove(const Key& key) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        return data_.erase(key) > 0;
    }

    // Writer: Clear all entries
    void clear() {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_.clear();
    }

    // Read-Modify-Write: Upgrade from shared to exclusive
    void update_if_exists(const Key& key,
                         std::function<Value(const Value&)> updater) {
        // Two-phase locking: read then write
        {
            std::shared_lock<std::shared_mutex> read_lock(mutex_);
            if (data_.find(key) == data_.end()) {
                return;  // Key doesn't exist
            }
        }

        // Upgrade to exclusive lock
        std::unique_lock<std::shared_mutex> write_lock(mutex_);
        auto it = data_.find(key);
        if (it != data_.end()) {
            it->second = updater(it->second);
        }
    }
};
```

### Complete Example: Cache System

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <random>

// Thread-safe cache with Reader-Writer pattern
template<typename K, typename V>
class Cache {
private:
    std::map<K, V> data_;
    mutable std::shared_mutex mutex_;

    // Statistics
    mutable std::atomic<size_t> hits_{0};
    mutable std::atomic<size_t> misses_{0};

public:
    // Read operation (shared lock)
    std::optional<V> lookup(const K& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);

        auto it = data_.find(key);
        if (it != data_.end()) {
            hits_.fetch_add(1, std::memory_order_relaxed);
            return it->second;
        }

        misses_.fetch_add(1, std::memory_order_relaxed);
        return std::nullopt;
    }

    // Write operation (exclusive lock)
    void insert(const K& key, const V& value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_[key] = value;
    }

    // Write operation: evict if cache is too large
    void evict_if_needed(size_t max_size) {
        std::unique_lock<std::shared_mutex> lock(mutex_);

        if (data_.size() > max_size) {
            // Simple eviction: remove first element
            data_.erase(data_.begin());
        }
    }

    void print_stats() const {
        size_t h = hits_.load();
        size_t m = misses_.load();
        size_t total = h + m;

        std::cout << "Cache Stats:\n"
                  << "  Hits: " << h << "\n"
                  << "  Misses: " << m << "\n"
                  << "  Hit Rate: "
                  << (total > 0 ? (100.0 * h / total) : 0) << "%\n";
    }
};

// Simulate cache workload
void reader_thread(const Cache<int, std::string>& cache, int id, int iterations) {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 99);

    for (int i = 0; i < iterations; ++i) {
        int key = dis(gen);
        auto value = cache.lookup(key);

        // Simulate some work
        std::this_thread::sleep_for(std::chrono::microseconds(10));
    }

    std::cout << "Reader " << id << " completed\n";
}

void writer_thread(Cache<int, std::string>& cache, int id, int iterations) {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 99);

    for (int i = 0; i < iterations; ++i) {
        int key = dis(gen);
        cache.insert(key, "value_" + std::to_string(key));

        // Simulate some work
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }

    std::cout << "Writer " << id << " completed\n";
}

int main() {
    Cache<int, std::string> cache;

    // Pre-populate cache
    for (int i = 0; i < 50; ++i) {
        cache.insert(i, "value_" + std::to_string(i));
    }

    const int NUM_READERS = 10;
    const int NUM_WRITERS = 2;
    const int ITERATIONS = 1000;

    std::vector<std::thread> threads;

    // Start readers (90% of workload)
    for (int i = 0; i < NUM_READERS; ++i) {
        threads.emplace_back(reader_thread, std::cref(cache), i, ITERATIONS);
    }

    // Start writers (10% of workload)
    for (int i = 0; i < NUM_WRITERS; ++i) {
        threads.emplace_back(writer_thread, std::ref(cache), i, ITERATIONS / 10);
    }

    // Wait for all threads
    for (auto& t : threads) {
        t.join();
    }

    cache.print_stats();

    return 0;
}
```

## Internal Mechanisms

### std::shared_mutex의 구현

Linux에서 std::shared_mutex는 pthread_rwlock을 래핑하며, 내부적으로 futex를 사용합니다.

#### 상태 인코딩

```c
// glibc pthread_rwlock 상태 (간략화)
struct pthread_rwlock_t {
    unsigned int __readers;
    // 비트 레이아웃:
    // [31]: WRPHASE (Writer가 락을 획득했거나 대기 중)
    // [30]: WRLOCKED (Writer가 락을 보유 중)
    // [29:0]: Reader 수

    unsigned int __writers_futex;  // Writer 대기용 futex
    unsigned int __readers_futex;  // Reader 대기용 futex

    // ...
};

/*
 * 상태 예시:
 *
 * 0x00000000: 락 해제됨, reader 없음
 * 0x00000003: 3개의 reader가 보유 중
 * 0xC0000000: Writer가 락 보유 중 (WRPHASE | WRLOCKED)
 * 0x80000002: Writer 대기 중, 2개 reader 보유 (WRPHASE만)
 */
```

#### Reader 락 획득 흐름

```c
int pthread_rwlock_rdlock(pthread_rwlock_t *rwlock) {
    unsigned int r;

retry:
    r = atomic_load(&rwlock->__readers);

    // Fast path: Writer 없고 reader 추가 가능
    if (!(r & (WRPHASE | WRLOCKED))) {
        if (atomic_compare_exchange_weak(&rwlock->__readers, &r, r + 1)) {
            return 0;  // 성공!
        }
        goto retry;
    }

    // Slow path: Writer가 있거나 대기 중
    // Reader는 futex에서 대기
    while (r & WRPHASE) {
        futex_wait(&rwlock->__readers_futex, ...);
        r = atomic_load(&rwlock->__readers);
    }

    goto retry;
}
```

#### Writer 락 획득 흐름

```c
int pthread_rwlock_wrlock(pthread_rwlock_t *rwlock) {
    unsigned int r;

    // 1단계: WRPHASE 플래그 설정 (새 reader 차단)
    r = atomic_load(&rwlock->__readers);
    while (!atomic_compare_exchange_weak(&rwlock->__readers, &r,
                                          r | WRPHASE)) {
        // CAS 실패, 재시도
    }

    // 2단계: 모든 기존 reader 종료 대기
    while ((r & READER_MASK) != 0) {
        futex_wait(&rwlock->__writers_futex, ...);
        r = atomic_load(&rwlock->__readers);
    }

    // 3단계: WRLOCKED 설정
    atomic_fetch_or(&rwlock->__readers, WRLOCKED);

    return 0;
}
```

### Lock Striping 기법

대규모 동시 접근을 위한 최적화입니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    Lock Striping                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  단일 RWLock:                                               │
│    모든 스레드가 하나의 락을 경쟁                            │
│    ┌────────────┐                                           │
│    │  RWLock    │◀─── Thread 1, 2, 3, 4, 5, 6 ...          │
│    └────────────┘                                           │
│                                                             │
│  Lock Striping (n개 락):                                    │
│    키의 해시에 따라 락 분산                                  │
│    ┌────────────┐                                           │
│    │ RWLock[0]  │◀─── Thread 1, 4 (hash % n == 0)          │
│    └────────────┘                                           │
│    ┌────────────┐                                           │
│    │ RWLock[1]  │◀─── Thread 2, 5 (hash % n == 1)          │
│    └────────────┘                                           │
│    ┌────────────┐                                           │
│    │ RWLock[2]  │◀─── Thread 3, 6 (hash % n == 2)          │
│    └────────────┘                                           │
│                                                             │
│  경합 감소: n배 (이상적인 경우)                              │
└─────────────────────────────────────────────────────────────┘
```

```cpp
// Lock Striping 구현 예시
template<typename K, typename V, size_t NumStripes = 16>
class StripedMap {
    struct Stripe {
        std::shared_mutex mutex;
        std::unordered_map<K, V> data;
    };

    std::array<Stripe, NumStripes> stripes_;

    size_t get_stripe(const K& key) {
        return std::hash<K>{}(key) % NumStripes;
    }

public:
    V get(const K& key) {
        size_t idx = get_stripe(key);
        std::shared_lock lock(stripes_[idx].mutex);
        return stripes_[idx].data.at(key);
    }

    void put(const K& key, const V& value) {
        size_t idx = get_stripe(key);
        std::unique_lock lock(stripes_[idx].mutex);
        stripes_[idx].data[key] = value;
    }
};
```

### 읽기 편향 워크로드 최적화

```cpp
// 스레드 로컬 읽기 카운터 (contention 감소)
class OptimizedRWLock {
    std::atomic<int> writer_active_{0};
    std::atomic<int> writer_waiting_{0};

    // 각 스레드별 로컬 읽기 카운터
    // 중앙 카운터 업데이트 빈도 감소
    struct alignas(64) LocalCounter {
        std::atomic<int> count{0};
    };
    std::array<LocalCounter, 64> local_readers_;

    int get_slot() {
        // 스레드 ID 기반 슬롯 선택
        return std::hash<std::thread::id>{}(
            std::this_thread::get_id()) % 64;
    }

public:
    void lock_shared() {
        int slot = get_slot();

        // 로컬 카운터 증가 (빠름, 경합 없음)
        local_readers_[slot].count.fetch_add(1, std::memory_order_acquire);

        // Writer 대기 확인
        if (writer_waiting_.load(std::memory_order_acquire) > 0 ||
            writer_active_.load(std::memory_order_acquire)) {
            // 느린 경로: Writer가 있으면 대기
            slow_path_read_lock();
        }
    }

    void unlock_shared() {
        int slot = get_slot();
        local_readers_[slot].count.fetch_sub(1, std::memory_order_release);
        // Writer가 대기 중이면 notify
    }

    int total_readers() {
        int sum = 0;
        for (auto& lc : local_readers_) {
            sum += lc.count.load(std::memory_order_relaxed);
        }
        return sum;
    }
};
```

## Advanced Implementation: Custom Reader-Writer Lock

### Reader-Preferred Lock

```cpp
#include <atomic>
#include <mutex>
#include <condition_variable>

class ReaderPreferredLock {
private:
    std::mutex mutex_;
    std::condition_variable reader_cv_;
    std::condition_variable writer_cv_;
    int readers_ = 0;        // Active readers
    bool writer_ = false;     // Active writer
    int waiting_writers_ = 0; // Waiting writers

public:
    void lock_shared() {  // Reader lock
        std::unique_lock<std::mutex> lock(mutex_);

        // Wait if there's an active writer
        reader_cv_.wait(lock, [this] { return !writer_; });

        ++readers_;
    }

    void unlock_shared() {  // Reader unlock
        std::unique_lock<std::mutex> lock(mutex_);

        --readers_;

        // If last reader, wake up a waiting writer
        if (readers_ == 0 && waiting_writers_ > 0) {
            writer_cv_.notify_one();
        }
    }

    void lock() {  // Writer lock
        std::unique_lock<std::mutex> lock(mutex_);

        ++waiting_writers_;

        // Wait for no readers and no active writer
        writer_cv_.wait(lock, [this] {
            return readers_ == 0 && !writer_;
        });

        --waiting_writers_;
        writer_ = true;
    }

    void unlock() {  // Writer unlock
        std::unique_lock<std::mutex> lock(mutex_);

        writer_ = false;

        // Wake up all waiting readers first (reader-preferred)
        reader_cv_.notify_all();

        // Then wake up one writer
        if (waiting_writers_ > 0) {
            writer_cv_.notify_one();
        }
    }
};
```

### Writer-Preferred Lock

```cpp
class WriterPreferredLock {
private:
    std::mutex mutex_;
    std::condition_variable cv_;
    int readers_ = 0;
    int writers_ = 0;
    int waiting_writers_ = 0;

public:
    void lock_shared() {
        std::unique_lock<std::mutex> lock(mutex_);

        // Wait if there are writers or waiting writers (writer-preferred)
        cv_.wait(lock, [this] {
            return writers_ == 0 && waiting_writers_ == 0;
        });

        ++readers_;
    }

    void unlock_shared() {
        std::unique_lock<std::mutex> lock(mutex_);
        --readers_;

        if (readers_ == 0) {
            cv_.notify_all();
        }
    }

    void lock() {
        std::unique_lock<std::mutex> lock(mutex_);

        ++waiting_writers_;
        cv_.wait(lock, [this] {
            return readers_ == 0 && writers_ == 0;
        });

        --waiting_writers_;
        ++writers_;
    }

    void unlock() {
        std::unique_lock<std::mutex> lock(mutex_);
        --writers_;
        cv_.notify_all();
    }
};
```

### Fair (FIFO) Reader-Writer Lock

```cpp
#include <queue>
#include <thread>

class FairReaderWriterLock {
private:
    struct Waiter {
        enum Type { READER, WRITER };
        Type type;
        std::condition_variable cv;
        bool notified = false;
    };

    std::mutex mutex_;
    std::queue<std::shared_ptr<Waiter>> queue_;
    int active_readers_ = 0;
    bool active_writer_ = false;

public:
    void lock_shared() {
        auto waiter = std::make_shared<Waiter>();
        waiter->type = Waiter::READER;

        std::unique_lock<std::mutex> lock(mutex_);
        queue_.push(waiter);

        waiter->cv.wait(lock, [&] {
            return waiter->notified;
        });

        ++active_readers_;
    }

    void unlock_shared() {
        std::unique_lock<std::mutex> lock(mutex_);
        --active_readers_;

        if (active_readers_ == 0) {
            notify_next();
        }
    }

    void lock() {
        auto waiter = std::make_shared<Waiter>();
        waiter->type = Waiter::WRITER;

        std::unique_lock<std::mutex> lock(mutex_);
        queue_.push(waiter);

        waiter->cv.wait(lock, [&] {
            return waiter->notified;
        });

        active_writer_ = true;
    }

    void unlock() {
        std::unique_lock<std::mutex> lock(mutex_);
        active_writer_ = false;
        notify_next();
    }

private:
    void notify_next() {
        while (!queue_.empty()) {
            auto waiter = queue_.front();

            if (waiter->type == Waiter::WRITER) {
                if (active_readers_ == 0 && !active_writer_) {
                    queue_.pop();
                    waiter->notified = true;
                    waiter->cv.notify_one();
                    break;
                } else {
                    break;  // Can't wake writer yet
                }
            } else {  // READER
                if (!active_writer_) {
                    queue_.pop();
                    waiter->notified = true;
                    waiter->cv.notify_one();
                    // Continue to wake more readers
                } else {
                    break;  // Can't wake readers while writer active
                }
            }
        }
    }
};
```

## RAII Guards

```cpp
// RAII shared lock guard
template<typename RWLock>
class SharedLockGuard {
private:
    RWLock& lock_;

public:
    explicit SharedLockGuard(RWLock& lock) : lock_(lock) {
        lock_.lock_shared();
    }

    ~SharedLockGuard() {
        lock_.unlock_shared();
    }

    SharedLockGuard(const SharedLockGuard&) = delete;
    SharedLockGuard& operator=(const SharedLockGuard&) = delete;
};

// RAII exclusive lock guard
template<typename RWLock>
class ExclusiveLockGuard {
private:
    RWLock& lock_;

public:
    explicit ExclusiveLockGuard(RWLock& lock) : lock_(lock) {
        lock_.lock();
    }

    ~ExclusiveLockGuard() {
        lock_.unlock();
    }

    ExclusiveLockGuard(const ExclusiveLockGuard&) = delete;
    ExclusiveLockGuard& operator=(const ExclusiveLockGuard&) = delete;
};

// Usage
void example(ReaderPreferredLock& rwlock, std::map<int, int>& data) {
    // Read operation
    {
        SharedLockGuard guard(rwlock);
        auto it = data.find(42);
        // ...
    }

    // Write operation
    {
        ExclusiveLockGuard guard(rwlock);
        data[42] = 100;
    }
}
```

## Performance Comparison

```cpp
#include <chrono>

template<typename Lock, typename Map>
void benchmark_rwlock(const std::string& name, int num_threads,
                      int read_ratio) {
    Lock rwlock;
    Map data;

    // Pre-populate
    for (int i = 0; i < 1000; ++i) {
        data[i] = i;
    }

    auto start = std::chrono::high_resolution_clock::now();

    std::vector<std::thread> threads;
    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([&, i] {
            std::random_device rd;
            std::mt19937 gen(rd());
            std::uniform_int_distribution<> dis(0, 99);

            for (int j = 0; j < 10000; ++j) {
                if (dis(gen) < read_ratio) {
                    // Read operation
                    SharedLockGuard guard(rwlock);
                    volatile auto val = data[dis(gen)];
                } else {
                    // Write operation
                    ExclusiveLockGuard guard(rwlock);
                    data[dis(gen)] = j;
                }
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << name << " (" << read_ratio << "% reads): "
              << ms << "ms\n";
}
```

## Common Pitfalls

### 1. Writer Starvation

```cpp
// BAD: Reader-preferred lock with continuous readers
// Writers may never get access

// GOOD: Use fair or writer-preferred lock
// Or implement write priority
```

### 2. Deadlock with Lock Upgrade

```cpp
// BAD: Try to upgrade from shared to exclusive
{
    std::shared_lock lock(mutex);
    // Read data

    // DEADLOCK: Can't upgrade!
    lock.unlock();
    std::unique_lock ulock(mutex);  // Other readers block this
}

// GOOD: Release shared lock first
{
    std::shared_lock lock(mutex);
    // Read data
}  // Release shared lock

{
    std::unique_lock lock(mutex);
    // Write data
}
```

### 3. Excessive Write Locking

```cpp
// BAD: Using write lock for read-modify-write
void increment(int key) {
    std::unique_lock lock(mutex_);
    data_[key]++;  // Could use atomic or optimistic locking
}

// BETTER: Consider lock-free or optimistic approaches
```

## Real-World Applications

### 1. Configuration Management
```cpp
// Many threads read config, rare updates
ConfigManager config;
auto value = config.get("database.host");  // Shared lock
config.set("database.host", "new_host");   // Exclusive lock
```

### 2. DNS Cache
```cpp
// Frequent lookups, infrequent updates
DNSCache cache;
auto ip = cache.resolve("example.com");  // Shared
cache.update("example.com", "1.2.3.4");  // Exclusive
```

### 3. Route Tables (Networking)
```cpp
// Packet forwarding reads routes constantly
// Rare route updates
RouteTable routes;
auto next_hop = routes.lookup(dest_ip);  // Shared
routes.add_route(network, gateway);      // Exclusive
```

### 4. In-Memory Databases
```cpp
// Read-heavy OLAP queries
// Occasional batch updates
Database db;
auto results = db.query("SELECT ...");  // Shared
db.bulk_insert(data);                   // Exclusive
```

## Variants

### 1. Upgradeable Reader-Writer Lock
Allows upgrading from read to write lock.

### 2. Sequence Lock (SeqLock)
Optimistic read lock for small data structures.

### 3. RCU (Read-Copy-Update)
Writers create copies, readers never block.

## Pros and Cons

### Pros
- Excellent read scalability (reads don't block each other)
- Maintains consistency for writers
- Better than mutex for read-heavy workloads
- Standard library support (std::shared_mutex)

### Cons
- More complex than simple mutex
- Write operations can be slower than mutex
- Risk of writer starvation (reader-preferred)
- Risk of reader starvation (writer-preferred)
- Overhead not worth it if reads and writes are balanced

## Best Practices

1. **Use when reads >> writes** (typically >90% reads)

2. **Choose fairness policy based on requirements**:
   - Reader-preferred: Maximum read throughput
   - Writer-preferred: Prevent writer starvation
   - Fair: Balanced, but more overhead

3. **Keep critical sections small**:
   ```cpp
   // Copy data out while holding lock
   auto copy = [&] {
       std::shared_lock lock(mutex);
       return data;  // Copy
   }();

   // Process without holding lock
   process(copy);
   ```

4. **Consider alternatives for frequent writes**:
   - Lock-free structures
   - Partitioned data (reduce contention)
   - Optimistic concurrency control

5. **Measure performance**: RW locks aren't always faster than mutex

## Summary

Reader-Writer locks are essential for:
- Read-heavy workloads
- Shared configuration and caches
- Maximizing read concurrency
- Maintaining write consistency

Choose the right variant (reader-preferred, writer-preferred, fair) based on your workload characteristics and fairness requirements.
