# Producer-Consumer 패턴

## 개요

Producer-Consumer 패턴은 가장 기본적인 동시성 패턴 중 하나입니다. 이 패턴은 데이터를 생성하는 thread(producer)와 데이터를 소비하는 thread(consumer)를 공유 버퍼 또는 큐를 사용하여 분리합니다. 이러한 분리를 통해 producer와 consumer가 서로 다른 속도로 동작할 수 있으며, 자연스러운 부하 분산을 제공합니다.

## 문제 정의

많은 애플리케이션에서:
- 데이터 producer와 consumer가 서로 다른 속도로 동작합니다
- producer와 consumer 간의 강한 결합은 바람직하지 않습니다
- 생산 또는 소비의 급증을 처리하기 위한 버퍼링이 필요합니다
- 여러 producer 및/또는 consumer가 동시에 작업해야 합니다

## 솔루션 아키텍처

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

### 핵심 구성 요소

1. **Producer**: 데이터 항목을 생성하여 큐에 추가합니다
2. **Consumer**: 큐에서 데이터 항목을 제거하고 처리합니다
3. **공유 큐**: 제한된 용량을 가진 thread-safe 버퍼입니다
4. **동기화**: 큐가 가득 찬 상태와 비어 있는 상태를 위한 condition variable을 사용합니다

## 기본 구현 (C++)

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

    // Producer: 블로킹 put
    bool put(T item) {
        std::unique_lock<std::mutex> lock(mutex_);

        // 큐가 가득 차지 않거나 닫힐 때까지 대기
        not_full_.wait(lock, [this] {
            return queue_.size() < capacity_ || closed_;
        });

        if (closed_) {
            return false;  // 큐가 닫혀 추가 불가
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();  // consumer를 깨움
        return true;
    }

    // Producer: 논블로킹 try_put
    bool try_put(T item) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (closed_ || queue_.size() >= capacity_) {
            return false;
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();
        return true;
    }

    // Producer: 타임아웃이 있는 put
    template<typename Rep, typename Period>
    bool put_for(T item, const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (!not_full_.wait_for(lock, timeout, [this] {
            return queue_.size() < capacity_ || closed_;
        })) {
            return false;  // 타임아웃
        }

        if (closed_) {
            return false;
        }

        queue_.push(std::move(item));
        not_empty_.notify_one();
        return true;
    }

    // Consumer: 블로킹 get
    std::optional<T> get() {
        std::unique_lock<std::mutex> lock(mutex_);

        // 큐가 비어 있지 않거나 닫힐 때까지 대기
        not_empty_.wait(lock, [this] {
            return !queue_.empty() || closed_;
        });

        if (queue_.empty()) {
            return std::nullopt;  // 큐가 닫혔고 비어 있음
        }

        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();  // producer를 깨움
        return item;
    }

    // Consumer: 논블로킹 try_get
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

    // Consumer: 타임아웃이 있는 get
    template<typename Rep, typename Period>
    std::optional<T> get_for(const std::chrono::duration<Rep, Period>& timeout) {
        std::unique_lock<std::mutex> lock(mutex_);

        if (!not_empty_.wait_for(lock, timeout, [this] {
            return !queue_.empty() || closed_;
        })) {
            return std::nullopt;  // 타임아웃
        }

        if (queue_.empty()) {
            return std::nullopt;  // 닫혔고 비어 있음
        }

        T item = std::move(queue_.front());
        queue_.pop();
        not_full_.notify_one();
        return item;
    }

    // 큐 닫기 (더 이상 항목을 추가할 수 없음)
    void close() {
        std::lock_guard<std::mutex> lock(mutex_);
        closed_ = true;
        not_full_.notify_all();   // 대기 중인 모든 producer를 깨움
        not_empty_.notify_all();  // 대기 중인 모든 consumer를 깨움
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

### 전체 예제: 이미지 처리 파이프라인

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <string>
#include <chrono>
#include <random>

// 시뮬레이션용 이미지 데이터
struct Image {
    int id;
    std::string filename;
    std::vector<uint8_t> data;

    Image(int id, const std::string& name)
        : id(id), filename(name), data(1024 * 1024) {}  // 1MB
};

// Producer: 디스크에서 이미지 로드
void image_loader(BoundedQueue<Image>& queue, int count) {
    std::cout << "로더 thread 시작\n";

    for (int i = 0; i < count; ++i) {
        // 디스크에서 이미지를 로드하는 것을 시뮬레이션
        std::this_thread::sleep_for(std::chrono::milliseconds(100));

        Image img(i, "image_" + std::to_string(i) + ".jpg");
        std::cout << "로드 완료: " << img.filename << "\n";

        if (!queue.put(std::move(img))) {
            std::cout << "큐가 닫힘, 로더 종료\n";
            break;
        }
    }

    std::cout << "로더 완료\n";
}

// Consumer: 이미지 처리
void image_processor(BoundedQueue<Image>& queue, int worker_id) {
    std::cout << "프로세서 " << worker_id << " 시작\n";

    while (true) {
        auto img = queue.get();

        if (!img.has_value()) {
            std::cout << "프로세서 " << worker_id << " 종료\n";
            break;
        }

        // 이미지 처리를 시뮬레이션
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
        std::cout << "프로세서 " << worker_id << " 처리 완료: "
                  << img->filename << "\n";
    }
}

int main() {
    const int NUM_IMAGES = 20;
    const int NUM_PROCESSORS = 3;
    const int QUEUE_CAPACITY = 5;

    BoundedQueue<Image> queue(QUEUE_CAPACITY);

    // Producer 시작
    std::thread loader(image_loader, std::ref(queue), NUM_IMAGES);

    // Consumer 시작
    std::vector<std::thread> processors;
    for (int i = 0; i < NUM_PROCESSORS; ++i) {
        processors.emplace_back(image_processor, std::ref(queue), i);
    }

    // Producer 완료 대기
    loader.join();

    // Consumer에게 종료 신호를 보내기 위해 큐 닫기
    queue.close();

    // 모든 consumer 완료 대기
    for (auto& p : processors) {
        p.join();
    }

    std::cout << "모든 처리 완료\n";
    return 0;
}
```

## 고급 변형: 우선순위가 있는 다중 큐

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

    // 우선순위를 지정하여 put (0 = 가장 높은 우선순위)
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

    // 가용한 가장 높은 우선순위 항목 가져오기
    std::optional<T> get() {
        std::unique_lock<std::mutex> lock(mutex_);
        not_empty_.wait(lock, [this] {
            return total_size_ > 0 || closed_;
        });

        if (total_size_ == 0) {
            return std::nullopt;
        }

        // 비어 있지 않은 가장 높은 우선순위 큐 찾기
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

## 내부 메커니즘

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

## Lock-Free 구현 (Single Producer, Single Consumer)

```cpp
#include <atomic>
#include <array>

template<typename T, size_t Capacity>
class SPSCQueue {
private:
    struct alignas(64) {  // 캐시 라인 정렬
        std::atomic<size_t> head{0};
    };
    struct alignas(64) {
        std::atomic<size_t> tail{0};
    };

    std::array<T, Capacity> buffer_;

public:
    // Producer 전용
    bool try_push(const T& item) {
        const size_t current_tail = tail.load(std::memory_order_relaxed);
        const size_t next_tail = (current_tail + 1) % Capacity;

        if (next_tail == head.load(std::memory_order_acquire)) {
            return false;  // 큐가 가득 참
        }

        buffer_[current_tail] = item;
        tail.store(next_tail, std::memory_order_release);
        return true;
    }

    // Consumer 전용
    bool try_pop(T& item) {
        const size_t current_head = head.load(std::memory_order_relaxed);

        if (current_head == tail.load(std::memory_order_acquire)) {
            return false;  // 큐가 비어 있음
        }

        item = buffer_[current_head];
        head.store((current_head + 1) % Capacity, std::memory_order_release);
        return true;
    }
};
```

## 성능 고려 사항

### 큐 용량 선택

```
너무 작은 경우 (capacity = 1-2):
  - 빈번한 블로킹
  - 낮은 처리량
  - 강한 결합

최적 (capacity = 10-100):
  - 급증 흡수
  - 좋은 처리량
  - 균형 잡힌 결합

너무 큰 경우 (capacity = 1000+):
  - 메모리 낭비
  - 낮은 캐시 지역성
  - 지연된 backpressure
```

### 벤치마킹 코드

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

    std::cout << "처리량: " << (num_items * 1000.0 / duration)
              << " items/sec\n";
}
```

## 흔한 실수

### 1. 큐 닫기를 잊는 경우

```cpp
// 잘못된 예: consumer가 영원히 대기함
queue.put(item);
// ... producer가 끝났지만 큐를 닫지 않음

// 올바른 예: 생산이 끝나면 항상 큐를 닫아야 함
queue.put(item);
queue.close();  // consumer에게 중지 신호를 보냄
```

### 2. 깨우기 손실 (Lost Wakeup)

```cpp
// 잘못된 예: lock 바깥에서 조건 확인
if (queue.empty()) {  // 경쟁 조건: 큐가 이미 비어 있지 않을 수 있음
    wait_on_condition_variable();
}

// 올바른 예: wait 술어 안에서 조건 확인
lock.wait([&]{ return !queue.empty(); });
```

### 3. Spurious Wakeup 미처리

```cpp
// 잘못된 예: 단일 확인
lock.wait();
auto item = queue.front();  // 여전히 비어 있을 수 있음!

// 올바른 예: 루프 또는 술어 사용
lock.wait([&]{ return !queue.empty(); });
auto item = queue.front();
```

## 변형 및 확장

### 1. Work Stealing 큐
여러 consumer가 서로의 로컬 큐에서 작업을 가져올 수 있습니다.

### 2. Buffered Channel (Go 스타일)
버퍼가 있는 통신과 버퍼가 없는 통신의 조합입니다.

### 3. Disruptor 패턴
저지연 시스템을 위한 초고성능 ring buffer입니다.

## 실제 활용 사례

### 1. Thread Pool 작업 큐
```
작업 producer → 작업 큐 → Worker thread
```

### 2. 메시지 브로커
```
발행자 → 메시지 큐 → 구독자
```

### 3. 데이터 처리 파이프라인
```
데이터 수집 → 처리 단계 → 출력
```

### 4. UI 이벤트 처리
```
사용자 이벤트 → 이벤트 큐 → 이벤트 처리 thread
```

### 5. 로그 집계
```
여러 서비스 → 로그 큐 → 로그 프로세서
```

## 장단점

### 장점
- producer와 consumer를 분리합니다
- consumer 간 자연스러운 부하 분산이 가능합니다
- 생산/소비 속도의 급증을 흡수합니다
- 이해하고 구현하기 쉽습니다
- thread pool과 잘 연동됩니다

### 단점
- bounded 큐는 producer 블로킹을 유발할 수 있습니다
- unbounded 큐는 메모리 고갈을 유발할 수 있습니다
- 큐 lock에 대한 높은 경합이 발생할 수 있습니다
- 요청-응답 패턴에는 적합하지 않습니다
- 큐 튜닝이 애플리케이션에 따라 달라질 수 있습니다

## 모범 사례

1. **적절한 큐 용량 선택** 시 고려 사항:
   - 항목 크기
   - 생산/소비 속도
   - 메모리 제약
   - 지연 시간 요구 사항

2. **프로덕션 시스템을 위한 타임아웃 변형 제공**:
   ```cpp
   if (!queue.put_for(item, 5s)) {
       // 타임아웃 처리
   }
   ```

3. **큐 깊이 모니터링**으로 다음을 감지:
   - Consumer 기아 상태
   - Producer 과부하
   - 시스템 불균형

4. **복사를 피하기 위해 이동 시맨틱스 사용**:
   ```cpp
   queue.put(std::move(large_object));
   ```

5. **다음 경우에 lock-free 구현 고려**:
   - Single producer, single consumer
   - 초저지연 요구 사항
   - 고빈도 거래

## 테스트 전략

### 스트레스 테스트
```cpp
// 많은 producer와 consumer로 테스트
const int NUM_PRODUCERS = 10;
const int NUM_CONSUMERS = 10;
const int ITEMS_PER_PRODUCER = 10000;
// 모든 항목이 정확히 한 번 생산되고 소비되는지 검증
```

### 공정성 테스트
```cpp
// consumer가 기아 상태에 빠지지 않는지 검증
// 부하가 consumer 간에 균형 있게 분배되는지 검증
```

### Backpressure 테스트
```cpp
// 느린 consumer, 빠른 producer
// producer가 적절히 블로킹되는지 검증
```

## 요약

Producer-Consumer 패턴은 다음에 필수적입니다:
- 시스템 구성 요소 분리
- 확장 가능한 데이터 처리 시스템 구축
- 가변적인 작업 부하 속도 처리
- 작업 분배 구현

이 패턴은 다른 많은 동시성 패턴의 기초를 이루며 실제 시스템에서 광범위하게 사용되므로 반드시 숙달해야 합니다.
