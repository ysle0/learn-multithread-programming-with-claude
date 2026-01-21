# Reader-Writer Lock (읽기-쓰기 락)

## 📌 핵심 개념

**Reader-Writer Lock (RWLock)**은 **읽기는 여러 스레드가 동시에**, **쓰기는 독점적으로** 수행하도록 하는 동기화 메커니즘입니다.

**핵심 원칙**:
- **여러 Reader 동시 허용**: 읽기 작업은 서로 간섭하지 않음
- **Writer 독점**: 쓰기 중에는 다른 Reader/Writer 불가
- **Reader-Writer 상호 배제**: 읽기와 쓰기는 동시에 불가

**장점**: 읽기가 많은 워크로드에서 성능 향상

---

## 🏗️ RWLock의 동작 원리

### 기본 개념

```
상태 1: 여러 Reader 동시 실행
┌─────────────────────────────────┐
│  Reader 1  Reader 2  Reader 3   │  ← 동시 읽기 OK
└─────────────────────────────────┘

상태 2: Writer 독점
┌─────────────────────────────────┐
│         Writer 1                │  ← 독점
│  [Reader 2 대기] ⏳             │
│  [Writer 2 대기] ⏳             │
└─────────────────────────────────┘

상태 3: Writer 완료 후 Reader들 진입
┌─────────────────────────────────┐
│  Reader 2  Reader 4  Reader 5   │  ← 동시 읽기 OK
│  [Writer 2 대기] ⏳             │
└─────────────────────────────────┘
```

### 상태 전이 다이어그램

```
        RWLock State Machine

    ┌──────────────┐
    │   Unlocked   │
    └──────┬───────┘
           │
      ┌────┴────┐
      │         │
  read_lock  write_lock
      │         │
      ↓         ↓
┌──────────┐  ┌──────────┐
│ Reading  │  │ Writing  │
│ (N개)    │  │ (1개)    │
└────┬─────┘  └────┬─────┘
     │             │
read_unlock   write_unlock
     │             │
     └─────┬───────┘
           ↓
    ┌──────────────┐
    │   Unlocked   │
    └──────────────┘
```

---

## 💻 C++ 구현

### 1. std::shared_mutex (C++17)

```cpp
#include <shared_mutex>
#include <thread>
#include <vector>
#include <iostream>

std::shared_mutex rw_mutex;
int shared_data = 0;

void reader(int id) {
    // 여러 Reader 동시 실행 가능
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    std::cout << "Reader " << id << " reads: " << shared_data << std::endl;
}

void writer(int id, int value) {
    // Writer는 독점
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    shared_data = value;
    std::cout << "Writer " << id << " writes: " << value << std::endl;
}

int main() {
    std::vector<std::thread> threads;

    // 5개 Reader (동시 실행)
    for (int i = 0; i < 5; i++) {
        threads.emplace_back(reader, i);
    }

    // 1개 Writer (독점)
    threads.emplace_back(writer, 0, 42);

    for (auto& t : threads) {
        t.join();
    }

    return 0;
}
```

### 2. std::shared_timed_mutex (타임아웃 지원)

```cpp
#include <shared_mutex>
#include <chrono>

std::shared_timed_mutex rw_mutex;

void try_read() {
    // 100ms 내에 읽기 락 획득 시도
    if (rw_mutex.try_lock_shared_for(std::chrono::milliseconds(100))) {
        // 읽기 수행
        rw_mutex.unlock_shared();
    } else {
        std::cout << "Read timeout!" << std::endl;
    }
}

void try_write() {
    // 500ms 내에 쓰기 락 획득 시도
    if (rw_mutex.try_lock_for(std::chrono::milliseconds(500))) {
        // 쓰기 수행
        rw_mutex.unlock();
    } else {
        std::cout << "Write timeout!" << std::endl;
    }
}
```

### 3. 직접 구현 (C++11, Mutex + Condition Variable)

```cpp
#include <mutex>
#include <condition_variable>

class RWLock {
private:
    std::mutex mtx;
    std::condition_variable cv;
    int readers = 0;
    bool writer = false;

public:
    void read_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return !writer; });
        readers++;
    }

    void read_unlock() {
        std::unique_lock<std::mutex> lock(mtx);
        readers--;
        if (readers == 0) {
            cv.notify_all();  // Writer 깨움
        }
    }

    void write_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return !writer && readers == 0; });
        writer = true;
    }

    void write_unlock() {
        std::unique_lock<std::mutex> lock(mtx);
        writer = false;
        cv.notify_all();  // 모든 대기자 깨움
    }
};

// RAII 래퍼
class ReadLock {
private:
    RWLock& rwlock;
public:
    explicit ReadLock(RWLock& rw) : rwlock(rw) { rwlock.read_lock(); }
    ~ReadLock() { rwlock.read_unlock(); }
};

class WriteLock {
private:
    RWLock& rwlock;
public:
    explicit WriteLock(RWLock& rw) : rwlock(rw) { rwlock.write_lock(); }
    ~WriteLock() { rwlock.write_unlock(); }
};
```

---

## 🎯 실전 사용 예시

### 1. 캐시 구현

```cpp
#include <shared_mutex>
#include <unordered_map>
#include <string>

class Cache {
private:
    std::shared_mutex rw_mutex;
    std::unordered_map<std::string, std::string> data;

public:
    // 읽기 (여러 스레드 동시 가능)
    std::string get(const std::string& key) {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        auto it = data.find(key);
        return (it != data.end()) ? it->second : "";
    }

    // 쓰기 (독점)
    void set(const std::string& key, const std::string& value) {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        data[key] = value;
    }

    // 삭제 (독점)
    void remove(const std::string& key) {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        data.erase(key);
    }

    // 크기 조회 (읽기)
    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        return data.size();
    }
};
```

### 2. 설정 관리자

```cpp
#include <shared_mutex>
#include <map>
#include <string>

class ConfigManager {
private:
    mutable std::shared_mutex rw_mutex;
    std::map<std::string, std::string> config;

public:
    // 읽기 (빈번함)
    std::string get_config(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        auto it = config.find(key);
        return (it != config.end()) ? it->second : "";
    }

    // 쓰기 (드물음)
    void update_config(const std::string& key, const std::string& value) {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        config[key] = value;
    }

    // 전체 설정 로드 (쓰기)
    void load_from_file(const std::string& filename) {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        // 파일에서 설정 로드
        config.clear();
        // ... 파일 읽기 ...
    }
};
```

### 3. 통계 수집기

```cpp
#include <shared_mutex>
#include <vector>

class Statistics {
private:
    mutable std::shared_mutex rw_mutex;
    std::vector<double> samples;
    double sum = 0.0;
    size_t count = 0;

public:
    // 쓰기 (빈번함)
    void add_sample(double value) {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        samples.push_back(value);
        sum += value;
        count++;
    }

    // 읽기 (빈번함)
    double get_average() const {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        return (count > 0) ? (sum / count) : 0.0;
    }

    // 읽기
    size_t get_count() const {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        return count;
    }

    // 쓰기 (드물음)
    void reset() {
        std::unique_lock<std::shared_mutex> lock(rw_mutex);
        samples.clear();
        sum = 0.0;
        count = 0;
    }
};
```

### 4. Lock Upgrade/Downgrade (주의!)

```cpp
// ⚠️ C++ std::shared_mutex는 upgrade/downgrade 미지원!
// 직접 구현 또는 외부 라이브러리 사용

class RWLockUpgradable {
    // Boost.Thread의 upgrade_lock 참고
    // pthread_rwlock도 upgrade 미지원
};

// 일반적인 해결책: unlock 후 다시 lock
std::shared_mutex rw_mutex;

void upgrade_example() {
    // Read lock
    rw_mutex.lock_shared();
    // 읽기 수행
    if (need_to_write) {
        rw_mutex.unlock_shared();  // Read unlock
        rw_mutex.lock();           // Write lock
        // 쓰기 수행
        rw_mutex.unlock();
    } else {
        rw_mutex.unlock_shared();
    }
}
```

---

## 🔍 정책: Reader-Preference vs Writer-Preference

### 1. Reader-Preference (읽기 우선)

```cpp
// Reader가 우선: 새로운 Reader는 대기 중인 Writer를 건너뜀

class ReaderPreferenceRWLock {
private:
    std::mutex mtx;
    std::condition_variable cv;
    int readers = 0;
    bool writer = false;

public:
    void read_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]{ return !writer; });
        readers++;
    }

    void write_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        // Reader가 모두 끝날 때까지 대기 (Starvation 가능!)
        cv.wait(lock, [this]{ return !writer && readers == 0; });
        writer = true;
    }
};
```

**장점**: 읽기 처리량 최대화
**단점**: Writer Starvation (Writer가 계속 대기)

### 2. Writer-Preference (쓰기 우선)

```cpp
class WriterPreferenceRWLock {
private:
    std::mutex mtx;
    std::condition_variable cv;
    int readers = 0;
    int writers_waiting = 0;
    bool writer = false;

public:
    void read_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        // Writer가 대기 중이면 새로운 Reader는 대기
        cv.wait(lock, [this]{ return !writer && writers_waiting == 0; });
        readers++;
    }

    void write_lock() {
        std::unique_lock<std::mutex> lock(mtx);
        writers_waiting++;
        cv.wait(lock, [this]{ return !writer && readers == 0; });
        writers_waiting--;
        writer = true;
    }
};
```

**장점**: Writer Starvation 방지
**단점**: 읽기 처리량 감소

### 3. Fair (공정한) RWLock

```cpp
// FIFO 순서로 락 획득 (구현 복잡)
// pthread_rwlock_t는 기본적으로 writer-preference
```

---

## 📊 성능 비교

### Mutex vs RWLock

```cpp
// 벤치마크: 90% 읽기, 10% 쓰기

// std::mutex: ~100ms
std::mutex mtx;
for (int i = 0; i < 1000000; i++) {
    std::lock_guard<std::mutex> lock(mtx);
    if (i % 10 == 0) data = i;  // 쓰기
    else int x = data;           // 읽기
}

// std::shared_mutex: ~20ms (5배 빠름!)
std::shared_mutex rw_mtx;
for (int i = 0; i < 1000000; i++) {
    if (i % 10 == 0) {
        std::unique_lock<std::shared_mutex> lock(rw_mtx);
        data = i;
    } else {
        std::shared_lock<std::shared_mutex> lock(rw_mtx);
        int x = data;
    }
}
```

### 읽기/쓰기 비율에 따른 성능

| 읽기:쓰기 | Mutex | RWLock | 배수 |
|-----------|-------|--------|------|
| **100:0** | 100ms | 10ms | 10x |
| **90:10** | 100ms | 20ms | 5x |
| **70:30** | 100ms | 50ms | 2x |
| **50:50** | 100ms | 80ms | 1.25x |
| **0:100** | 100ms | 120ms | 0.8x (느림!) |

**결론**: 읽기가 70% 이상일 때 효과적

---

## ⚠️ 주의사항 및 함정

### 1. Writer Starvation

```cpp
// ❌ 문제: Reader-Preference에서 Writer가 계속 대기
std::shared_mutex rw_mutex;

// 계속해서 Reader가 진입
void continuous_readers() {
    while (true) {
        std::shared_lock<std::shared_mutex> lock(rw_mutex);
        read_data();
    }
}

// Writer는 영원히 대기!
void starving_writer() {
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    write_data();  // 영원히 도달 못함!
}
```

**해결책**: Writer-Preference RWLock 사용 또는 타임아웃

### 2. Deadlock (Upgrade)

```cpp
// ❌ 나쁜 예: Read lock에서 Write lock으로 upgrade
std::shared_mutex rw_mutex;

void deadlock_example() {
    std::shared_lock<std::shared_mutex> read_lock(rw_mutex);
    // 읽기 수행

    if (need_to_write) {
        // Deadlock! (이미 shared_lock 보유)
        std::unique_lock<std::shared_mutex> write_lock(rw_mutex);
    }
}
```

```cpp
// ✅ 좋은 예: Unlock 후 다시 lock
void correct_upgrade() {
    {
        std::shared_lock<std::shared_mutex> read_lock(rw_mutex);
        // 읽기 수행
    }  // unlock

    if (need_to_write) {
        std::unique_lock<std::shared_mutex> write_lock(rw_mutex);
        // 쓰기 수행 (데이터 재확인 필요!)
    }
}
```

### 3. 쓰기가 많을 때 비효율

```cpp
// ❌ 쓰기가 50% 이상이면 Mutex보다 느림!
std::shared_mutex rw_mutex;

for (int i = 0; i < 1000000; i++) {
    std::unique_lock<std::shared_mutex> lock(rw_mutex);
    write_data();  // 계속 쓰기
}
// shared_mutex 오버헤드로 인해 느림
```

**교훈**: 읽기:쓰기 비율 > 7:3일 때만 사용

### 4. 재진입 불가

```cpp
// ❌ 재진입 Deadlock
std::shared_mutex rw_mutex;

void inner() {
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    // ...
}

void outer() {
    std::shared_lock<std::shared_mutex> lock(rw_mutex);
    inner();  // Deadlock! (구현에 따라)
}
```

**참고**: `std::shared_mutex`는 재진입 미지원 (일반 `std::mutex`처럼)

---

## 🔍 고급 주제

### 1. Optimistic Read (낙관적 읽기)

```cpp
// Java의 StampedLock 개념
class OptimisticRWLock {
private:
    std::atomic<long> version{0};
    std::mutex write_mtx;

public:
    long try_optimistic_read() {
        return version.load(std::memory_order_acquire);
    }

    bool validate(long stamp) {
        return version.load(std::memory_order_acquire) == stamp;
    }

    void write_lock() {
        write_mtx.lock();
    }

    void write_unlock() {
        version.fetch_add(1, std::memory_order_release);
        write_mtx.unlock();
    }
};

// 사용
OptimisticRWLock lock;
int data;

void optimistic_read() {
    long stamp = lock.try_optimistic_read();
    int value = data;  // 락 없이 읽기

    if (!lock.validate(stamp)) {
        // 쓰기가 발생했으면 재시도
    }
}
```

### 2. Scalable RWLock (분산 RWLock)

```cpp
// 각 Reader가 개별 카운터를 가짐 (캐시 라인 분리)
class ScalableRWLock {
private:
    struct alignas(64) ReaderCount {
        std::atomic<int> count{0};
    };

    std::vector<ReaderCount> reader_counts;
    std::mutex write_mtx;

public:
    ScalableRWLock(int num_threads) : reader_counts(num_threads) {}

    void read_lock(int thread_id) {
        reader_counts[thread_id].count.fetch_add(1);
    }

    void read_unlock(int thread_id) {
        reader_counts[thread_id].count.fetch_sub(1);
    }

    void write_lock() {
        write_mtx.lock();
        // 모든 Reader가 끝날 때까지 대기
        for (auto& rc : reader_counts) {
            while (rc.count.load() > 0);
        }
    }
};
```

---

## 🔗 다음 단계

- [Mutex와 Lock](./01-mutex-lock.md) - 기본 동기화 복습
- [Atomic Operations](./04-atomic-operations.md) - Lock-Free 대안
- [동시성 패턴](../04-concurrency-patterns/README.md) - 실전 패턴

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams, Chapter 3
- [cppreference: shared_mutex](https://en.cppreference.com/w/cpp/thread/shared_mutex)
- [POSIX: pthread_rwlock](https://man7.org/linux/man-pages/man3/pthread_rwlock_rdlock.3p.html)
- [Java StampedLock](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/StampedLock.html)

---

*Reader-Writer Lock은 읽기가 많은 워크로드에서 성능을 크게 향상시킵니다. 읽기:쓰기 비율을 확인하고 사용하세요!*
