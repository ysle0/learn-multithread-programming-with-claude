# 동시성 코드 테스팅 전략

## 📌 개요

동시성 코드는 비결정적(non-deterministic) 동작으로 인해 테스트가 매우 어렵습니다. 이 문서에서는 효과적인 테스팅 전략과 기법을 소개합니다.

---

## 🎯 동시성 테스트의 과제

### 주요 문제점

1. **비결정성**: 실행마다 결과가 다를 수 있음
2. **타이밍 의존**: 미묘한 타이밍에 따라 버그 발생
3. **Heisenbug**: 관찰하려 하면 사라지는 버그
4. **낮은 재현율**: 버그가 드물게 발생
5. **상태 공간 폭발**: 가능한 인터리빙의 수가 기하급수적 증가

### 테스트 목표

1. **정확성**: Race Condition, Deadlock 없음
2. **활성성**: 진전(Progress) 보장
3. **성능**: 확장성 및 처리량
4. **안정성**: 장시간 실행 시 안정적

---

## 🧪 테스트 전략

### 1. 단위 테스트 (Unit Testing)

#### 기본 원칙

```cpp
// 테스트 가능한 설계
class ThreadSafeCounter {
public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_);
        ++counter_;
    }

    int get() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return counter_;
    }

    // 테스트용: 락 상태 확인
    bool is_locked_for_testing() const {
        return !mtx_.try_lock();
    }

private:
    mutable std::mutex mtx_;
    int counter_ = 0;
};

// 단위 테스트
TEST(ThreadSafeCounter, Increment) {
    ThreadSafeCounter counter;

    counter.increment();
    EXPECT_EQ(1, counter.get());

    counter.increment();
    EXPECT_EQ(2, counter.get());
}
```

#### 멀티스레드 단위 테스트

```cpp
#include <gtest/gtest.h>
#include <thread>
#include <vector>

TEST(ThreadSafeCounter, ConcurrentIncrement) {
    ThreadSafeCounter counter;
    const int NUM_THREADS = 10;
    const int INCREMENTS_PER_THREAD = 1000;

    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back([&counter]() {
            for (int j = 0; j < INCREMENTS_PER_THREAD; ++j) {
                counter.increment();
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    // 정확한 값 검증
    EXPECT_EQ(NUM_THREADS * INCREMENTS_PER_THREAD, counter.get());
}
```

---

### 2. 스트레스 테스트 (Stress Testing)

#### 고부하 테스트

```cpp
TEST(ThreadSafeQueue, StressTest) {
    ThreadSafeQueue<int> queue;
    const int NUM_PRODUCERS = 8;
    const int NUM_CONSUMERS = 8;
    const int ITEMS_PER_PRODUCER = 10000;

    std::atomic<int> produced{0};
    std::atomic<int> consumed{0};

    std::vector<std::thread> threads;

    // Producers
    for (int i = 0; i < NUM_PRODUCERS; ++i) {
        threads.emplace_back([&queue, &produced, i]() {
            for (int j = 0; j < ITEMS_PER_PRODUCER; ++j) {
                queue.push(i * ITEMS_PER_PRODUCER + j);
                produced.fetch_add(1, std::memory_order_relaxed);
            }
        });
    }

    // Consumers
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        threads.emplace_back([&queue, &consumed, total = NUM_PRODUCERS * ITEMS_PER_PRODUCER]() {
            while (consumed.load(std::memory_order_relaxed) < total) {
                int value;
                if (queue.try_pop(value)) {
                    consumed.fetch_add(1, std::memory_order_relaxed);
                } else {
                    std::this_thread::yield();
                }
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    EXPECT_EQ(NUM_PRODUCERS * ITEMS_PER_PRODUCER, consumed.load());
    EXPECT_TRUE(queue.empty());
}
```

#### 장시간 실행 테스트

```cpp
TEST(ThreadPool, LongRunningStability) {
    ThreadPool pool(4);
    const auto duration = std::chrono::minutes(5);
    const auto start = std::chrono::steady_clock::now();

    std::atomic<int> tasks_completed{0};

    while (std::chrono::steady_clock::now() - start < duration) {
        pool.submit([&tasks_completed]() {
            // 작업 수행
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
            tasks_completed.fetch_add(1, std::memory_order_relaxed);
        });
    }

    pool.wait_for_all_tasks();

    // 충분한 작업이 완료되었는지 확인
    EXPECT_GT(tasks_completed.load(), 10000);
}
```

---

### 3. Property-Based Testing

#### 속성 정의

```cpp
// Hypothesis: 큐는 FIFO 순서 유지
TEST(ThreadSafeQueue, FIFOProperty) {
    ThreadSafeQueue<int> queue;

    // 순차적으로 삽입
    for (int i = 0; i < 100; ++i) {
        queue.push(i);
    }

    // 순차적으로 추출
    for (int i = 0; i < 100; ++i) {
        int value;
        ASSERT_TRUE(queue.try_pop(value));
        EXPECT_EQ(i, value);  // FIFO 검증
    }
}
```

#### 불변식(Invariant) 검증

```cpp
class BoundedQueue {
public:
    explicit BoundedQueue(size_t capacity) : capacity_(capacity) {}

    bool push(int value) {
        std::lock_guard<std::mutex> lock(mtx_);

        if (queue_.size() >= capacity_) {
            return false;
        }

        queue_.push(value);

        // 불변식: 크기는 항상 capacity 이하
        assert(queue_.size() <= capacity_);
        return true;
    }

    bool pop(int& value) {
        std::lock_guard<std::mutex> lock(mtx_);

        if (queue_.empty()) {
            return false;
        }

        value = queue_.front();
        queue_.pop();

        // 불변식: 크기는 항상 0 이상
        assert(queue_.size() >= 0);
        return true;
    }

private:
    std::mutex mtx_;
    std::queue<int> queue_;
    size_t capacity_;
};

TEST(BoundedQueue, InvariantHolds) {
    BoundedQueue queue(10);

    std::vector<std::thread> threads;

    for (int i = 0; i < 20; ++i) {
        threads.emplace_back([&queue, i]() {
            for (int j = 0; j < 100; ++j) {
                queue.push(i * 100 + j);

                int value;
                queue.pop(value);
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    // 불변식이 항상 유지되었으면 여기까지 도달
    SUCCEED();
}
```

---

### 4. Model Checking

#### Deterministic Execution

```cpp
// 실행 순서를 제어하는 테스트
TEST(LockFreeStack, DeterministicInterleaving) {
    LockFreeStack<int> stack;

    std::barrier sync_point(2);

    std::thread t1([&]() {
        stack.push(1);
        sync_point.arrive_and_wait();  // 동기화 지점
        stack.push(2);
    });

    std::thread t2([&]() {
        sync_point.arrive_and_wait();  // 동기화 지점
        int value;
        bool success = stack.try_pop(value);
        EXPECT_TRUE(success);
        EXPECT_EQ(1, value);
    });

    t1.join();
    t2.join();
}
```

---

### 5. Fuzzing (Fuzz Testing)

#### 랜덤 입력 생성

```cpp
#include <random>

TEST(ConcurrentHashMap, FuzzTest) {
    ConcurrentHashMap<int, int> map;

    std::random_device rd;
    std::mt19937 gen(rd());
    std::uniform_int_distribution<> key_dist(0, 99);
    std::uniform_int_distribution<> op_dist(0, 2);  // 0: insert, 1: find, 2: erase

    std::vector<std::thread> threads;

    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([&]() {
            for (int j = 0; j < 1000; ++j) {
                int key = key_dist(gen);
                int op = op_dist(gen);

                switch (op) {
                case 0:  // Insert
                    map.insert(key, key * 2);
                    break;
                case 1:  // Find
                    {
                        int value;
                        map.find(key, value);
                    }
                    break;
                case 2:  // Erase
                    map.erase(key);
                    break;
                }
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    // 크래시 없이 완료되면 성공
    SUCCEED();
}
```

---

## 🔧 테스팅 도구

### 1. Google Test (C++)

#### 설치

```bash
# CMake
find_package(GTest REQUIRED)
target_link_libraries(my_test GTest::GTest GTest::Main)
```

#### 사용 예제

```cpp
#include <gtest/gtest.h>

class ConcurrentQueueTest : public ::testing::Test {
protected:
    void SetUp() override {
        queue = std::make_unique<ThreadSafeQueue<int>>();
    }

    void TearDown() override {
        queue.reset();
    }

    std::unique_ptr<ThreadSafeQueue<int>> queue;
};

TEST_F(ConcurrentQueueTest, BasicOperations) {
    queue->push(42);

    int value;
    ASSERT_TRUE(queue->try_pop(value));
    EXPECT_EQ(42, value);
}

TEST_F(ConcurrentQueueTest, ConcurrentOperations) {
    // 동시성 테스트...
}
```

---

### 2. Go Testing

#### 테이블 기반 테스트

```go
package concurrent

import (
    "sync"
    "testing"
)

func TestSafeCounter(t *testing.T) {
    tests := []struct {
        name       string
        goroutines int
        increments int
        want       int
    }{
        {"Single", 1, 100, 100},
        {"Dual", 2, 100, 200},
        {"Many", 10, 1000, 10000},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            counter := NewSafeCounter()
            var wg sync.WaitGroup

            for i := 0; i < tt.goroutines; i++ {
                wg.Add(1)
                go func() {
                    defer wg.Done()
                    for j := 0; j < tt.increments; j++ {
                        counter.Inc()
                    }
                }()
            }

            wg.Wait()

            if got := counter.Value(); got != tt.want {
                t.Errorf("Value() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

#### 벤치마크

```go
func BenchmarkSafeCounter(b *testing.B) {
    counter := NewSafeCounter()

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            counter.Inc()
        }
    })
}

// 실행
// go test -bench=. -race
```

---

### 3. Criterion (Rust)

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use std::sync::Arc;
use std::thread;

fn concurrent_increment(c: &mut Criterion) {
    c.bench_function("concurrent_increment", |b| {
        b.iter(|| {
            let counter = Arc::new(AtomicCounter::new());
            let handles: Vec<_> = (0..4)
                .map(|_| {
                    let counter = Arc::clone(&counter);
                    thread::spawn(move || {
                        for _ in 0..1000 {
                            counter.increment();
                        }
                    })
                })
                .collect();

            for h in handles {
                h.join().unwrap();
            }

            assert_eq!(counter.get(), 4000);
        });
    });
}

criterion_group!(benches, concurrent_increment);
criterion_main!(benches);
```

---

## 📊 테스트 커버리지

### 1. 코드 커버리지

```bash
# C++ (gcov)
g++ -fprofile-arcs -ftest-coverage test.cpp -o test
./test
gcov test.cpp

# Go
go test -cover
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

### 2. 동시성 커버리지

모든 가능한 인터리빙을 테스트하기는 불가능하므로:

1. **중요 시나리오 선택**: 가능성 높은 인터리빙
2. **스트레스 테스트**: 많은 스레드로 다양한 조합 유도
3. **도구 활용**: TSan, Race Detector로 검증

---

## 🎓 베스트 프랙티스

### 1. 테스트 격리

```cpp
// ❌ 나쁜 예: 전역 상태 공유
static ThreadPool global_pool(4);

TEST(ThreadPool, Test1) {
    global_pool.submit(task1);
    // 다른 테스트의 영향 받을 수 있음
}

// ✅ 좋은 예: 테스트마다 새 인스턴스
TEST(ThreadPool, Test1) {
    ThreadPool pool(4);
    pool.submit(task1);
    // 독립적
}
```

### 2. 타임아웃 설정

```cpp
TEST(SlowOperation, WithTimeout) {
    auto future = std::async(std::launch::async, []() {
        // 느린 작업
        std::this_thread::sleep_for(std::chrono::seconds(10));
    });

    // 5초 타임아웃
    auto status = future.wait_for(std::chrono::seconds(5));
    EXPECT_EQ(std::future_status::timeout, status);
}
```

### 3. Flaky Test 회피

```cpp
// ❌ Flaky: 타이밍 의존
TEST(FlakyTest, BadExample) {
    std::thread t([]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        set_flag();
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    EXPECT_FALSE(check_flag());  // Race condition!

    t.join();
}

// ✅ 안정적: 동기화 사용
TEST(StableTest, GoodExample) {
    std::promise<void> promise;
    auto future = promise.get_future();

    std::thread t([&promise]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        set_flag();
        promise.set_value();
    });

    EXPECT_FALSE(check_flag());

    future.wait();  // 확실히 대기
    EXPECT_TRUE(check_flag());

    t.join();
}
```

### 4. 재현 가능한 테스트

```cpp
// 시드 고정
TEST(RandomTest, Reproducible) {
    std::mt19937 gen(12345);  // 고정된 시드
    std::uniform_int_distribution<> dist(0, 99);

    // 항상 같은 결과
    EXPECT_EQ(42, dist(gen));
}
```

---

## 🔍 실전 예제

### Producer-Consumer 테스트

```cpp
class ProducerConsumerTest : public ::testing::Test {
protected:
    void SetUp() override {
        queue = std::make_unique<BlockingQueue<int>>(100);
    }

    std::unique_ptr<BlockingQueue<int>> queue;
};

TEST_F(ProducerConsumerTest, SingleProducerSingleConsumer) {
    const int COUNT = 1000;
    std::vector<int> consumed;
    std::mutex consumed_mtx;

    std::thread producer([this]() {
        for (int i = 0; i < COUNT; ++i) {
            queue->push(i);
        }
    });

    std::thread consumer([this, &consumed, &consumed_mtx]() {
        for (int i = 0; i < COUNT; ++i) {
            int value = queue->pop();
            std::lock_guard<std::mutex> lock(consumed_mtx);
            consumed.push_back(value);
        }
    });

    producer.join();
    consumer.join();

    // 검증
    EXPECT_EQ(COUNT, consumed.size());

    std::sort(consumed.begin(), consumed.end());
    for (int i = 0; i < COUNT; ++i) {
        EXPECT_EQ(i, consumed[i]);
    }
}

TEST_F(ProducerConsumerTest, MultipleProducersMultipleConsumers) {
    const int NUM_PRODUCERS = 4;
    const int NUM_CONSUMERS = 4;
    const int ITEMS_PER_PRODUCER = 250;

    std::vector<int> consumed;
    std::mutex consumed_mtx;

    std::vector<std::thread> threads;

    // Producers
    for (int i = 0; i < NUM_PRODUCERS; ++i) {
        threads.emplace_back([this, i]() {
            for (int j = 0; j < ITEMS_PER_PRODUCER; ++j) {
                queue->push(i * ITEMS_PER_PRODUCER + j);
            }
        });
    }

    // Consumers
    std::atomic<int> consumed_count{0};
    for (int i = 0; i < NUM_CONSUMERS; ++i) {
        threads.emplace_back([this, &consumed, &consumed_mtx, &consumed_count]() {
            while (consumed_count.load() < NUM_PRODUCERS * ITEMS_PER_PRODUCER) {
                try {
                    int value = queue->pop_with_timeout(std::chrono::milliseconds(100));

                    std::lock_guard<std::mutex> lock(consumed_mtx);
                    consumed.push_back(value);
                    consumed_count.fetch_add(1);
                } catch (const TimeoutException&) {
                    // 타임아웃, 재시도
                }
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    // 검증: 모든 아이템이 정확히 한 번씩 소비됨
    EXPECT_EQ(NUM_PRODUCERS * ITEMS_PER_PRODUCER, consumed.size());

    std::sort(consumed.begin(), consumed.end());
    for (int i = 0; i < NUM_PRODUCERS * ITEMS_PER_PRODUCER; ++i) {
        EXPECT_EQ(i, consumed[i]);
    }
}
```

---

## 📚 추가 리소스

### 도구
- Google Test: https://github.com/google/googletest
- Catch2: https://github.com/catchorg/Catch2
- Criterion (Rust): https://github.com/bheisler/criterion.rs

### 논문
- "Effective Testing of Concurrent Programs" (Lu et al.)
- "Finding and Reproducing Heisenbugs in Concurrent Programs" (Musuvathi et al.)

### 책
- "The Art of Unit Testing" (Roy Osherove)
- "Working Effectively with Legacy Code" (Michael Feathers)

---

*다음: [성능 튜닝](./performance-tuning.md)*
