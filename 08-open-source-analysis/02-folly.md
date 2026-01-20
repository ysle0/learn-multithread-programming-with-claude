# Facebook Folly - C++ 라이브러리

## 📌 프로젝트 개요

**Folly (Facebook Open-source LibrarY)**는 Facebook이 개발하고 사용하는 고성능 C++ 컴포넌트 라이브러리입니다.

### 기본 정보
- **저장소**: https://github.com/facebook/folly
- **언어**: C++14/17/20
- **라이선스**: Apache License 2.0
- **주요 사용처**: Facebook, Meta 서비스
- **활성 상태**: 매우 활발히 개발 중
- **난이도**: ⭐⭐⭐⭐

### 주요 특징
- 프로덕션에서 검증된 고성능 컴포넌트
- 광범위한 동시성 지원
- 풍부한 문서화와 테스트
- 현대적인 C++ 스타일
- 크로스 플랫폼 (Linux 중심, macOS, Windows 일부 지원)

---

## 🎯 왜 Folly를 분석해야 하는가?

### 학습 가치

1. **프로덕션 레벨 코드**
   - Facebook의 대규모 서비스에서 사용
   - 수십억 사용자의 트래픽 처리
   - 철저한 테스트와 최적화

2. **실전 동시성 패턴**
   - Lock-Free 자료구조
   - Future/Promise 비동기 프로그래밍
   - ThreadPool 설계

3. **성능 최적화의 정수**
   - Cache-aware 설계
   - SIMD 활용
   - Zero-copy 기법

4. **현대적인 C++ 활용**
   - Template metaprogramming
   - C++14/17/20 기능 적극 활용
   - Type-safe 인터페이스

---

## 🏗️ 아키텍처

### 전체 구조

```
folly/
├── concurrency/           # 동시성 프리미티브
│   ├── CacheLocality.h   # CPU 캐시 최적화
│   ├── ConcurrentHashMap.h
│   └── UnboundedQueue.h
├── executors/            # Executor 프레임워크
│   ├── ThreadPoolExecutor.h
│   ├── CPUThreadPoolExecutor.h
│   └── IOThreadPoolExecutor.h
├── futures/              # Future/Promise
│   ├── Future.h
│   ├── Promise.h
│   └── SharedPromise.h
├── synchronization/      # 동기화 도구
│   ├── Baton.h          # 이벤트 동기화
│   ├── DistributedMutex.h
│   └── RWSpinLock.h
└── AtomicHashMap.h      # Lock-Free HashMap
```

### 핵심 컴포넌트

#### 1. MPMCQueue - Multi-Producer Multi-Consumer Queue
```cpp
#include <folly/MPMCQueue.h>

folly::MPMCQueue<int> queue(1024);  // 용량 지정 필수

// Producer
queue.write(42);

// Consumer
int value;
queue.read(value);
```

#### 2. Future/Promise - 비동기 프로그래밍
```cpp
#include <folly/futures/Future.h>

folly::Future<int> async_computation()
{
    folly::Promise<int> promise;
    auto future = promise.getFuture();

    std::thread([p = std::move(promise)]() mutable {
        // 비동기 작업
        p.setValue(42);
    }).detach();

    return future;
}
```

#### 3. ThreadPoolExecutor
```cpp
#include <folly/executors/CPUThreadPoolExecutor.h>

folly::CPUThreadPoolExecutor executor(4);  // 4 threads

executor.add([]() {
    // 작업 수행
});
```

---

## 🔬 핵심 컴포넌트 분석

### 1. MPMCQueue (Multi-Producer Multi-Consumer Queue)

#### 개요
Bounded, blocking MPMC queue로 매우 높은 성능을 자랑합니다.

#### 핵심 설계

**턴 기반 시퀀싱 (Turn-based sequencing)**

```cpp
template <typename T>
class MPMCQueue
{
private:
    struct Slot {
        std::atomic<uint64_t> turn;  // 턴 번호
        T data;

        Slot() : turn(0) {}
    };

    const size_t capacity_;
    const size_t stride_;  // False sharing 방지
    Slot* slots_;

    std::atomic<uint64_t> pushTicket_;  // Producer 티켓
    std::atomic<uint64_t> popTicket_;   // Consumer 티켓

public:
    explicit MPMCQueue(size_t capacity)
        : capacity_(capacity)
        , stride_(compute_stride())
        , slots_(new Slot[capacity * stride_])
        , pushTicket_(0)
        , popTicket_(0)
    {
        // 각 슬롯의 턴 초기화
        for (size_t i = 0; i < capacity_; ++i) {
            slots_[idx(i)].turn.store(i, std::memory_order_relaxed);
        }
    }

    void write(T const& value)
    {
        // 1. 티켓 획득
        uint64_t ticket = pushTicket_.fetch_add(1, std::memory_order_relaxed);
        Slot& slot = slots_[idx(ticket)];

        // 2. 자신의 턴이 될 때까지 대기
        uint64_t turn = ticket;
        while (slot.turn.load(std::memory_order_acquire) != turn) {
            // Busy-wait with exponential backoff
            std::this_thread::yield();
        }

        // 3. 데이터 쓰기
        slot.data = value;

        // 4. 다음 턴으로 진행 (consumer가 읽을 수 있도록)
        slot.turn.store(turn + capacity_, std::memory_order_release);
    }

    void read(T& dest)
    {
        // 1. 티켓 획득
        uint64_t ticket = popTicket_.fetch_add(1, std::memory_order_relaxed);
        Slot& slot = slots_[idx(ticket)];

        // 2. 데이터가 준비될 때까지 대기
        uint64_t turn = ticket + capacity_;
        while (slot.turn.load(std::memory_order_acquire) != turn) {
            std::this_thread::yield();
        }

        // 3. 데이터 읽기
        dest = slot.data;

        // 4. 다음 턴으로 진행 (producer가 쓸 수 있도록)
        slot.turn.store(turn + capacity_, std::memory_order_release);
    }

private:
    size_t idx(uint64_t ticket) const
    {
        return (ticket % capacity_) * stride_;
    }

    size_t compute_stride() const
    {
        // False sharing 방지를 위해 cache line 크기로 정렬
        return (sizeof(Slot) + 63) / 64;
    }
};
```

#### 핵심 아이디어

**1. 티켓 기반 순서 보장**
```cpp
uint64_t ticket = pushTicket_.fetch_add(1);
```
- Fetch-and-Add로 순서 보장
- Wait-Free 티켓 발급

**2. 턴 기반 대기**
```cpp
while (slot.turn.load() != my_turn) {
    yield();
}
```
- 각 슬롯은 턴 번호로 상태 관리
- Producer와 Consumer가 교대로 접근

**3. False Sharing 방지**
```cpp
const size_t stride_ = cache_line_size / sizeof(Slot);
```
- 슬롯을 캐시 라인 크기로 정렬
- 성능 크게 향상

#### 성능 특성
- **처리량**: 스레드당 수천만 ops/sec
- **레이턴시**: 수십 나노초
- **확장성**: 거의 선형 확장
- **공정성**: 완벽한 FIFO 순서 보장

---

### 2. Future/Promise 프레임워크

#### 개요
비동기 프로그래밍을 위한 강력한 추상화입니다.

#### 기본 구조

```cpp
template <typename T>
class Future
{
private:
    std::shared_ptr<Core<T>> core_;

public:
    // Continuation 체인
    template <typename F>
    auto then(F&& func) -> Future<decltype(func(std::declval<T>()))>
    {
        using R = decltype(func(std::declval<T>()));
        Promise<R> promise;
        auto future = promise.getFuture();

        setCallback([
            promise = std::move(promise),
            func = std::forward<F>(func)
        ](T value) mutable {
            try {
                promise.setValue(func(std::move(value)));
            } catch (...) {
                promise.setException(std::current_exception());
            }
        });

        return future;
    }

    // Error handling
    Future<T> onError(std::function<T(std::exception_ptr)> func)
    {
        Promise<T> promise;
        auto future = promise.getFuture();

        setCallback([
            promise = std::move(promise),
            func = std::move(func)
        ](Try<T> t) mutable {
            if (t.hasException()) {
                try {
                    promise.setValue(func(t.exception()));
                } catch (...) {
                    promise.setException(std::current_exception());
                }
            } else {
                promise.setValue(std::move(t.value()));
            }
        });

        return future;
    }

    // Blocking wait
    T get()
    {
        wait();
        return std::move(core_->value_);
    }

    void wait()
    {
        core_->wait();
    }
};

template <typename T>
class Promise
{
private:
    std::shared_ptr<Core<T>> core_;

public:
    void setValue(T value)
    {
        core_->setValue(std::move(value));
    }

    void setException(std::exception_ptr e)
    {
        core_->setException(std::move(e));
    }

    Future<T> getFuture()
    {
        return Future<T>(core_);
    }
};
```

#### 실전 사용 예제

**1. 체이닝 (Chaining)**
```cpp
folly::Future<int> compute()
{
    return folly::makeFuture(42)
        .then([](int x) { return x * 2; })
        .then([](int x) { return x + 10; })
        .then([](int x) {
            std::cout << "Result: " << x << std::endl;
            return x;
        });
}

// Result: 94
```

**2. 에러 처리**
```cpp
folly::Future<std::string> fetchUser(int userId)
{
    return folly::makeFuture()
        .then([userId]() {
            if (userId < 0) {
                throw std::runtime_error("Invalid user ID");
            }
            return getUserFromDB(userId);
        })
        .onError([](std::exception const& e) {
            std::cerr << "Error: " << e.what() << std::endl;
            return std::string("Guest");
        });
}
```

**3. 병렬 실행 후 수집**
```cpp
std::vector<folly::Future<int>> futures;

for (int i = 0; i < 10; ++i) {
    futures.push_back(
        folly::async([i]() { return expensive_computation(i); })
    );
}

folly::Future<std::vector<int>> all = folly::collectAll(futures)
    .then([](std::vector<folly::Try<int>> results) {
        std::vector<int> values;
        for (auto& result : results) {
            values.push_back(result.value());
        }
        return values;
    });

std::vector<int> results = all.get();
```

**4. Timeout 처리**
```cpp
folly::Future<int> withTimeout()
{
    return slow_operation()
        .within(std::chrono::seconds(5))
        .onError([](folly::FutureTimeout const&) {
            std::cerr << "Operation timed out!" << std::endl;
            return -1;
        });
}
```

#### 장점
- **조합 가능**: 여러 비동기 작업을 쉽게 조합
- **타입 안전**: 컴파일 타임 타입 체크
- **에러 전파**: 예외가 자동으로 전파됨
- **성능**: 최소한의 오버헤드

---

### 3. ThreadPoolExecutor

#### 개요
작업 스케줄링을 위한 고성능 스레드 풀입니다.

#### 주요 타입

**1. CPUThreadPoolExecutor**
```cpp
#include <folly/executors/CPUThreadPoolExecutor.h>

// CPU-bound 작업용
folly::CPUThreadPoolExecutor executor(
    8,                           // 스레드 개수
    std::make_shared<folly::LifoSemMPMCQueue<
        folly::CPUThreadPoolExecutor::CPUTask>>(1024)
);

executor.add([]() {
    // CPU 집약적 작업
    heavy_computation();
});
```

**2. IOThreadPoolExecutor**
```cpp
#include <folly/executors/IOThreadPoolExecutor.h>

// I/O-bound 작업용 (event loop 기반)
folly::IOThreadPoolExecutor io_executor(4);

io_executor.add([]() {
    // I/O 작업
    read_from_network();
});
```

#### 고급 기능

**1. 동적 스레드 풀 크기**
```cpp
executor.setNumThreads(16);  // 런타임에 조정
```

**2. 작업 우선순위**
```cpp
executor.addWithPriority([]() {
    critical_task();
}, folly::Executor::HI_PRI);

executor.addWithPriority([]() {
    background_task();
}, folly::Executor::LO_PRI);
```

**3. Future와 통합**
```cpp
folly::Future<int> future = folly::via(&executor, []() {
    return compute();
}).then([](int result) {
    return result * 2;
});
```

**4. 통계 및 모니터링**
```cpp
auto stats = executor.getPoolStats();
std::cout << "Active threads: " << stats.activeThreadCount << std::endl;
std::cout << "Pending tasks: " << stats.pendingTaskCount << std::endl;
```

---

### 4. ConcurrentHashMap

#### 개요
Lock-Free에 가까운 성능의 동시성 해시맵입니다.

#### 기본 사용법

```cpp
#include <folly/concurrency/ConcurrentHashMap.h>

folly::ConcurrentHashMap<int, std::string> map;

// 삽입
map.insert(1, "one");
map.insert_or_assign(2, "two");

// 조회
auto it = map.find(1);
if (it != map.end()) {
    std::cout << it->second << std::endl;
}

// 업데이트
map.assign_if_equal(1, "ONE", "one");  // CAS 기반 업데이트

// 삭제
map.erase(2);

// 원자적 업데이트
map.insert_or_assign(3, "three");
```

#### 고급 사용법

**1. 원자적 조작**
```cpp
// 값이 없으면 삽입, 있으면 업데이트
map.emplace_or_visit(
    key,
    []() { return initial_value(); },  // 삽입할 값 생성
    [](std::string& value) {            // 기존 값 수정
        value += " updated";
    }
);
```

**2. 일괄 조회**
```cpp
std::vector<int> keys = {1, 2, 3, 4, 5};
for (int key : keys) {
    map.find_fn(key, [](const auto& value) {
        std::cout << value << std::endl;
    });
}
```

#### 내부 구조

```cpp
// 간소화된 버전
template <typename K, typename V>
class ConcurrentHashMap
{
private:
    struct Node {
        K key;
        V value;
        std::atomic<Node*> next;
    };

    struct Segment {
        std::mutex mutex;  // 세그먼트별 락
        std::atomic<Node*> head;
    };

    std::vector<Segment> segments_;

    Segment& get_segment(K const& key)
    {
        size_t hash = std::hash<K>{}(key);
        return segments_[hash % segments_.size()];
    }

public:
    void insert(K key, V value)
    {
        auto& segment = get_segment(key);
        std::lock_guard<std::mutex> lock(segment.mutex);

        // 링크드 리스트에 삽입
        Node* node = new Node{key, value, segment.head.load()};
        segment.head.store(node);
    }

    bool find(K const& key, V& value)
    {
        auto& segment = get_segment(key);

        // Lock-free 읽기
        Node* node = segment.head.load(std::memory_order_acquire);
        while (node) {
            if (node->key == key) {
                value = node->value;
                return true;
            }
            node = node->next.load(std::memory_order_acquire);
        }
        return false;
    }
};
```

#### 성능 특성
- **읽기**: Lock-Free, 매우 빠름
- **쓰기**: 세그먼트별 락, 높은 동시성
- **확장성**: 거의 선형
- **메모리**: 오버헤드 낮음

---

### 5. Baton - 경량 이벤트 동기화

#### 개요
`std::condition_variable`보다 훨씬 빠른 경량 동기화 프리미티브입니다.

#### 사용법

```cpp
#include <folly/synchronization/Baton.h>

folly::Baton<> baton;

// 대기 스레드
std::thread waiter([&]() {
    baton.wait();  // 시그널까지 대기
    std::cout << "Signaled!" << std::endl;
});

// 작업 수행
std::this_thread::sleep_for(std::chrono::seconds(1));

// 시그널
baton.post();

waiter.join();
```

#### 고급 기능

**1. Timeout 대기**
```cpp
if (baton.try_wait_for(std::chrono::seconds(5))) {
    std::cout << "Received signal" << std::endl;
} else {
    std::cout << "Timeout" << std::endl;
}
```

**2. 재사용 불가 (일회용)**
```cpp
folly::Baton<> baton;
baton.post();
baton.wait();  // 즉시 반환
// baton.reset();  // Error: 재사용 불가
```

#### 내부 구조

```cpp
template <bool MayBlock = true>
class Baton
{
private:
    std::atomic<uint32_t> state_;
    // MayBlock = true일 때 futex 사용

public:
    void post()
    {
        if (state_.exchange(1, std::memory_order_release) == 0) {
            if (MayBlock) {
                futex_wake();  // 대기 중인 스레드 깨우기
            }
        }
    }

    void wait()
    {
        if (state_.load(std::memory_order_acquire) == 1) {
            return;  // 이미 시그널됨
        }

        if (MayBlock) {
            while (state_.load(std::memory_order_acquire) == 0) {
                futex_wait();  // 커널에서 대기
            }
        } else {
            // Spin-wait
            while (state_.load(std::memory_order_acquire) == 0) {
                std::this_thread::yield();
            }
        }
    }
};
```

#### 성능 비교
```
Baton:                 ~20ns (fast path)
std::condition_variable: ~500ns
```

---

## 🚀 실전 예제

### 비동기 HTTP 서버

```cpp
#include <folly/futures/Future.h>
#include <folly/executors/CPUThreadPoolExecutor.h>
#include <folly/io/async/AsyncServerSocket.h>

class AsyncHTTPServer
{
private:
    folly::CPUThreadPoolExecutor executor_;

public:
    AsyncHTTPServer() : executor_(8) {}

    folly::Future<std::string> handleRequest(std::string path)
    {
        return folly::via(&executor_, [path]() {
            // CPU-bound 처리
            return processPath(path);
        })
        .then([](std::string result) {
            // I/O-bound 처리
            return fetchFromDB(result);
        })
        .onError([](std::exception const& e) {
            return std::string("Error: ") + e.what();
        })
        .within(std::chrono::seconds(5));
    }

private:
    static std::string processPath(std::string const& path)
    {
        // 경로 파싱 등
        return path;
    }

    static folly::Future<std::string> fetchFromDB(std::string const& key)
    {
        // 비동기 DB 조회
        return folly::makeFuture(std::string("Data for ") + key);
    }
};
```

### Producer-Consumer 패턴

```cpp
#include <folly/MPMCQueue.h>

template <typename T>
class ProducerConsumer
{
private:
    folly::MPMCQueue<T> queue_;
    std::atomic<bool> stopped_{false};

public:
    explicit ProducerConsumer(size_t capacity) : queue_(capacity) {}

    void produce(T item)
    {
        queue_.blockingWrite(std::move(item));
    }

    std::optional<T> consume()
    {
        T item;
        if (queue_.read(item)) {
            return item;
        }
        return std::nullopt;
    }

    void stop()
    {
        stopped_.store(true, std::memory_order_release);
    }

    bool is_stopped() const
    {
        return stopped_.load(std::memory_order_acquire);
    }
};

// 사용 예제
void example()
{
    ProducerConsumer<int> pc(1024);

    // Producer 스레드
    std::thread producer([&]() {
        for (int i = 0; i < 10000; ++i) {
            pc.produce(i);
        }
        pc.stop();
    });

    // Consumer 스레드들
    std::vector<std::thread> consumers;
    for (int i = 0; i < 4; ++i) {
        consumers.emplace_back([&]() {
            while (!pc.is_stopped()) {
                if (auto item = pc.consume()) {
                    process(*item);
                } else {
                    std::this_thread::yield();
                }
            }
        });
    }

    producer.join();
    for (auto& t : consumers) {
        t.join();
    }
}
```

---

## 📊 성능 최적화 기법

### 1. Cache-Aware 설계

```cpp
// Cache line 크기 고려
struct alignas(64) CacheAlignedCounter {
    std::atomic<uint64_t> value;
    char padding[64 - sizeof(std::atomic<uint64_t>)];
};

CacheAlignedCounter counters[8];  // False sharing 없음
```

### 2. Memory Ordering 최적화

```cpp
// Relaxed ordering for counters
counter_.fetch_add(1, std::memory_order_relaxed);

// Acquire-Release for synchronization
data_ = new_data;
ready_.store(true, std::memory_order_release);

// Consumer
while (!ready_.load(std::memory_order_acquire)) {
    // wait
}
use(data_);
```

### 3. Hazard Pointer 최적화

Folly는 자체 Hazard Pointer 구현을 가지고 있습니다:

```cpp
#include <folly/synchronization/Hazptr.h>

struct Node : public folly::hazptr_obj_base<Node> {
    int data;
    std::atomic<Node*> next;
};

folly::hazptr_holder h;  // Hazard Pointer 홀더
Node* node = h.protect(head_);  // 노드 보호
// 안전하게 node 사용
```

---

## 🎓 학습 포인트

### 1. 빌드 및 설치

```bash
# Ubuntu/Debian
sudo apt-get install \
    g++ \
    cmake \
    libboost-all-dev \
    libevent-dev \
    libdouble-conversion-dev \
    libgoogle-glog-dev \
    libgflags-dev \
    libiberty-dev \
    liblz4-dev \
    liblzma-dev \
    libsnappy-dev \
    make \
    zlib1g-dev \
    binutils-dev \
    libjemalloc-dev \
    libssl-dev \
    pkg-config \
    libunwind-dev

# Folly 빌드
git clone https://github.com/facebook/folly.git
cd folly
mkdir build && cd build
cmake ..
make -j$(nproc)
sudo make install
```

### 2. 간단한 예제

```cpp
#include <folly/MPMCQueue.h>
#include <folly/futures/Future.h>
#include <iostream>

int main()
{
    // MPMCQueue 예제
    folly::MPMCQueue<int> queue(10);
    queue.blockingWrite(42);

    int value;
    queue.blockingRead(value);
    std::cout << "Read: " << value << std::endl;

    // Future 예제
    auto future = folly::makeFuture(100)
        .then([](int x) { return x * 2; })
        .then([](int x) { return x + 50; });

    std::cout << "Result: " << future.get() << std::endl;

    return 0;
}
```

### 3. 벤치마크

```cpp
#include <folly/Benchmark.h>
#include <folly/MPMCQueue.h>

BENCHMARK(MPMCQueue_Write_Read, n)
{
    folly::MPMCQueue<int> queue(1024);

    for (unsigned i = 0; i < n; ++i) {
        queue.blockingWrite(i);
        int value;
        queue.blockingRead(value);
    }
}

BENCHMARK_RELATIVE(StdQueue_Write_Read, n)
{
    std::queue<int> queue;
    std::mutex mtx;

    for (unsigned i = 0; i < n; ++i) {
        {
            std::lock_guard<std::mutex> lock(mtx);
            queue.push(i);
        }
        {
            std::lock_guard<std::mutex> lock(mtx);
            queue.pop();
        }
    }
}

int main(int argc, char** argv)
{
    folly::runBenchmarks();
    return 0;
}
```

---

## ⚠️ 주의사항

### 1. 플랫폼 의존성
- Linux에서 가장 잘 지원됨
- macOS는 일부 기능 제한
- Windows는 제한적 지원

### 2. 의존성 관리
- 많은 외부 라이브러리 필요
- Boost, glog, gflags 등

### 3. 컴파일 시간
- 템플릿 헤비한 라이브러리
- 컴파일 시간이 길 수 있음

---

## 🔗 관련 리소스

### 공식 리소스
- GitHub: https://github.com/facebook/folly
- 문서: https://github.com/facebook/folly/tree/main/folly/docs

### 추천 글
- "Building Fast Interpreters in Rust" (Folly 영감)
- CppCon 발표: "Futures, async, and coroutines"

### 관련 프로젝트
- **libcds**: Lock-Free 자료구조
- **Seastar**: 비동기 C++ 프레임워크
- **Abseil**: Google C++ 라이브러리

---

## 📚 다음 단계

1. **Folly 설치 및 빌드**
2. **MPMCQueue 예제 실행**
3. **Future/Promise 패턴 학습**
4. **실제 프로젝트에 적용**
5. **다음: [Nakama](./03-nakama.md) 게임 서버 분석**

---

*이 문서는 학습 목적으로 작성되었습니다. Folly 라이선스를 확인하시기 바랍니다.*
