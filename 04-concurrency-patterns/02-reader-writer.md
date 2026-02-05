# Reader-Writer 패턴

## 개요

Reader-Writer 패턴(Shared-Exclusive Lock 패턴이라고도 함)은 읽기가 쓰기보다 압도적으로 많은 경우 공유 데이터에 대한 동시 접근을 최적화합니다. 여러 reader가 동시에 데이터에 접근할 수 있도록 허용하면서 writer에게는 독점적인 접근을 보장합니다. 이 패턴은 고성능 동시성 데이터 구조를 구축하는 데 핵심적입니다.

## 문제 정의

많은 시스템에서:
- 읽기 연산이 쓰기 연산보다 훨씬 많음 (90% 이상이 읽기)
- 여러 reader가 안전하게 동시에 데이터에 접근 가능
- Writer는 일관성을 유지하기 위해 독점적 접근이 필요
- 단순한 상호 배제(mutex)는 모든 접근을 직렬화하여 동시성 잠재력을 낭비

## 솔루션 아키텍처

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

## 기본 구현 (C++17)

### std::shared_mutex 사용

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
    // Reader: shared lock 획득
    std::optional<Value> get(const Key& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);

        auto it = data_.find(key);
        if (it != data_.end()) {
            return it->second;
        }
        return std::nullopt;
    }

    // Reader: 키 존재 여부 확인
    bool contains(const Key& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        return data_.find(key) != data_.end();
    }

    // Reader: 크기 조회
    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        return data_.size();
    }

    // Writer: exclusive lock 획득
    void put(const Key& key, const Value& value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_[key] = value;
    }

    // Writer: 항목 제거
    bool remove(const Key& key) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        return data_.erase(key) > 0;
    }

    // Writer: 모든 항목 제거
    void clear() {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_.clear();
    }

    // Read-Modify-Write: shared에서 exclusive로 업그레이드
    void update_if_exists(const Key& key,
                         std::function<Value(const Value&)> updater) {
        // 2단계 잠금: 먼저 읽기 후 쓰기
        {
            std::shared_lock<std::shared_mutex> read_lock(mutex_);
            if (data_.find(key) == data_.end()) {
                return;  // 키가 존재하지 않음
            }
        }

        // exclusive lock으로 업그레이드
        std::unique_lock<std::shared_mutex> write_lock(mutex_);
        auto it = data_.find(key);
        if (it != data_.end()) {
            it->second = updater(it->second);
        }
    }
};
```

### 완전한 예제: 캐시 시스템

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <random>

// Reader-Writer 패턴을 사용한 thread 안전 캐시
template<typename K, typename V>
class Cache {
private:
    std::map<K, V> data_;
    mutable std::shared_mutex mutex_;

    // 통계
    mutable std::atomic<size_t> hits_{0};
    mutable std::atomic<size_t> misses_{0};

public:
    // 읽기 연산 (shared lock)
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

    // 쓰기 연산 (exclusive lock)
    void insert(const K& key, const V& value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_[key] = value;
    }

    // 쓰기 연산: 캐시가 너무 클 경우 제거
    void evict_if_needed(size_t max_size) {
        std::unique_lock<std::shared_mutex> lock(mutex_);

        if (data_.size() > max_size) {
            // 단순 제거: 첫 번째 요소 제거
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

// 캐시 워크로드 시뮬레이션
void reader_thread(const Cache<int, std::string>& cache, int id, int iterations) {
    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> dis(0, 99);

    for (int i = 0; i < iterations; ++i) {
        int key = dis(gen);
        auto value = cache.lookup(key);

        // 작업 시뮬레이션
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

        // 작업 시뮬레이션
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
    }

    std::cout << "Writer " << id << " completed\n";
}

int main() {
    Cache<int, std::string> cache;

    // 캐시 사전 채우기
    for (int i = 0; i < 50; ++i) {
        cache.insert(i, "value_" + std::to_string(i));
    }

    const int NUM_READERS = 10;
    const int NUM_WRITERS = 2;
    const int ITERATIONS = 1000;

    std::vector<std::thread> threads;

    // Reader 시작 (워크로드의 90%)
    for (int i = 0; i < NUM_READERS; ++i) {
        threads.emplace_back(reader_thread, std::cref(cache), i, ITERATIONS);
    }

    // Writer 시작 (워크로드의 10%)
    for (int i = 0; i < NUM_WRITERS; ++i) {
        threads.emplace_back(writer_thread, std::ref(cache), i, ITERATIONS / 10);
    }

    // 모든 thread 대기
    for (auto& t : threads) {
        t.join();
    }

    cache.print_stats();

    return 0;
}
```

## 내부 메커니즘

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

## 고급 구현: 커스텀 Reader-Writer Lock

### Reader 우선 Lock

```cpp
#include <atomic>
#include <mutex>
#include <condition_variable>

class ReaderPreferredLock {
private:
    std::mutex mutex_;
    std::condition_variable reader_cv_;
    std::condition_variable writer_cv_;
    int readers_ = 0;        // 활성 reader
    bool writer_ = false;     // 활성 writer
    int waiting_writers_ = 0; // 대기 중인 writer

public:
    void lock_shared() {  // Reader lock
        std::unique_lock<std::mutex> lock(mutex_);

        // 활성 writer가 있으면 대기
        reader_cv_.wait(lock, [this] { return !writer_; });

        ++readers_;
    }

    void unlock_shared() {  // Reader unlock
        std::unique_lock<std::mutex> lock(mutex_);

        --readers_;

        // 마지막 reader이면 대기 중인 writer 깨우기
        if (readers_ == 0 && waiting_writers_ > 0) {
            writer_cv_.notify_one();
        }
    }

    void lock() {  // Writer lock
        std::unique_lock<std::mutex> lock(mutex_);

        ++waiting_writers_;

        // reader 없고 활성 writer 없을 때까지 대기
        writer_cv_.wait(lock, [this] {
            return readers_ == 0 && !writer_;
        });

        --waiting_writers_;
        writer_ = true;
    }

    void unlock() {  // Writer unlock
        std::unique_lock<std::mutex> lock(mutex_);

        writer_ = false;

        // 대기 중인 모든 reader를 먼저 깨움 (reader 우선)
        reader_cv_.notify_all();

        // 그 다음 writer 하나 깨움
        if (waiting_writers_ > 0) {
            writer_cv_.notify_one();
        }
    }
};
```

### Writer 우선 Lock

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

        // writer 또는 대기 중인 writer가 있으면 대기 (writer 우선)
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

### 공정한 (FIFO) Reader-Writer Lock

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
                    break;  // 아직 writer를 깨울 수 없음
                }
            } else {  // READER
                if (!active_writer_) {
                    queue_.pop();
                    waiter->notified = true;
                    waiter->cv.notify_one();
                    // 더 많은 reader를 계속 깨움
                } else {
                    break;  // writer가 활성 상태인 동안 reader를 깨울 수 없음
                }
            }
        }
    }
};
```

## RAII 가드

```cpp
// RAII shared lock 가드
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

// RAII exclusive lock 가드
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

// 사용 예시
void example(ReaderPreferredLock& rwlock, std::map<int, int>& data) {
    // 읽기 연산
    {
        SharedLockGuard guard(rwlock);
        auto it = data.find(42);
        // ...
    }

    // 쓰기 연산
    {
        ExclusiveLockGuard guard(rwlock);
        data[42] = 100;
    }
}
```

## 성능 비교

```cpp
#include <chrono>

template<typename Lock, typename Map>
void benchmark_rwlock(const std::string& name, int num_threads,
                      int read_ratio) {
    Lock rwlock;
    Map data;

    // 사전 채우기
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
                    // 읽기 연산
                    SharedLockGuard guard(rwlock);
                    volatile auto val = data[dis(gen)];
                } else {
                    // 쓰기 연산
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

## 흔한 함정

### 1. Writer 기아 현상

```cpp
// 나쁜 예: 지속적인 reader가 있는 Reader 우선 lock
// Writer가 영원히 접근하지 못할 수 있음

// 좋은 예: 공정한 lock 또는 Writer 우선 lock 사용
// 또는 쓰기 우선 순위 구현
```

### 2. Lock 업그레이드 시 교착 상태

```cpp
// 나쁜 예: shared에서 exclusive로 업그레이드 시도
{
    std::shared_lock lock(mutex);
    // 데이터 읽기

    // 교착 상태: 업그레이드 불가!
    lock.unlock();
    std::unique_lock ulock(mutex);  // 다른 reader가 이것을 차단
}

// 좋은 예: shared lock을 먼저 해제
{
    std::shared_lock lock(mutex);
    // 데이터 읽기
}  // shared lock 해제

{
    std::unique_lock lock(mutex);
    // 데이터 쓰기
}
```

### 3. 과도한 쓰기 잠금

```cpp
// 나쁜 예: read-modify-write에 쓰기 lock 사용
void increment(int key) {
    std::unique_lock lock(mutex_);
    data_[key]++;  // atomic 또는 낙관적 잠금을 사용할 수 있음
}

// 더 나은 예: lock-free 또는 낙관적 접근 방식 고려
```

## 실제 응용 사례

### 1. 설정 관리
```cpp
// 많은 thread가 설정을 읽고, 드물게 업데이트
ConfigManager config;
auto value = config.get("database.host");  // Shared lock
config.set("database.host", "new_host");   // Exclusive lock
```

### 2. DNS 캐시
```cpp
// 빈번한 조회, 드문 업데이트
DNSCache cache;
auto ip = cache.resolve("example.com");  // Shared
cache.update("example.com", "1.2.3.4");  // Exclusive
```

### 3. 라우팅 테이블 (네트워킹)
```cpp
// 패킷 포워딩이 라우팅을 지속적으로 읽음
// 드문 라우팅 업데이트
RouteTable routes;
auto next_hop = routes.lookup(dest_ip);  // Shared
routes.add_route(network, gateway);      // Exclusive
```

### 4. 인메모리 데이터베이스
```cpp
// 읽기 집중 OLAP 쿼리
// 간헐적인 배치 업데이트
Database db;
auto results = db.query("SELECT ...");  // Shared
db.bulk_insert(data);                   // Exclusive
```

## 변형

### 1. 업그레이드 가능한 Reader-Writer Lock
읽기 lock에서 쓰기 lock으로 업그레이드 가능합니다.

### 2. Sequence Lock (SeqLock)
작은 데이터 구조를 위한 낙관적 읽기 lock입니다.

### 3. RCU (Read-Copy-Update)
Writer가 복사본을 생성하고, reader는 절대 차단되지 않습니다.

## 장단점

### 장점
- 뛰어난 읽기 확장성 (읽기가 서로를 차단하지 않음)
- Writer에 대한 일관성 유지
- 읽기 집중 워크로드에서 mutex보다 우수
- 표준 라이브러리 지원 (std::shared_mutex)

### 단점
- 단순 mutex보다 복잡
- 쓰기 연산이 mutex보다 느릴 수 있음
- Writer 기아 위험 (reader 우선)
- Reader 기아 위험 (writer 우선)
- 읽기와 쓰기가 균형을 이루면 오버헤드가 가치 없음

## 모범 사례

1. **읽기가 쓰기보다 훨씬 많을 때 사용** (일반적으로 90% 이상 읽기)

2. **요구 사항에 따라 공정성 정책 선택**:
   - Reader 우선: 최대 읽기 처리량
   - Writer 우선: Writer 기아 방지
   - 공정: 균형적이지만 오버헤드가 더 큼

3. **임계 구역을 작게 유지**:
   ```cpp
   // lock을 보유한 상태에서 데이터를 복사하여 꺼냄
   auto copy = [&] {
       std::shared_lock lock(mutex);
       return data;  // 복사
   }();

   // lock을 보유하지 않은 상태에서 처리
   process(copy);
   ```

4. **빈번한 쓰기 시 대안 고려**:
   - Lock-free 구조
   - 분할된 데이터 (경합 감소)
   - 낙관적 동시성 제어

5. **성능 측정**: RW lock이 항상 mutex보다 빠른 것은 아님

## 요약

Reader-Writer lock은 다음에 필수적입니다:
- 읽기 집중 워크로드
- 공유 설정 및 캐시
- 읽기 동시성 극대화
- 쓰기 일관성 유지

워크로드 특성과 공정성 요구 사항에 따라 적절한 변형(reader 우선, writer 우선, 공정)을 선택하십시오.
