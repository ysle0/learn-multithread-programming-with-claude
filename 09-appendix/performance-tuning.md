# 성능 튜닝 가이드

## 📌 개요

멀티스레드 프로그램의 성능을 최적화하는 것은 복잡한 작업입니다. 이 문서에서는 체계적인 성능 튜닝 방법과 실전 기법을 소개합니다.

---

## 🎯 성능 튜닝의 원칙

### 기본 원칙

1. **측정 먼저**: "추측하지 말고 측정하라"
2. **병목 식별**: 가장 큰 영향을 주는 부분 찾기
3. **점진적 개선**: 한 번에 하나씩 변경
4. **회귀 테스트**: 변경 후 항상 측정

### 최적화 순서

```
1. 알고리즘 개선 (가장 큰 영향)
   ↓
2. 동시성 모델 선택
   ↓
3. 자료구조 최적화
   ↓
4. 캐시 효율성
   ↓
5. 미세 최적화 (마지막)
```

---

## 📊 성능 측정

### 1. 벤치마킹

#### Google Benchmark (C++)

```cpp
#include <benchmark/benchmark.h>
#include <mutex>
#include <shared_mutex>

std::mutex mtx;
std::shared_mutex sh_mtx;
int shared_data = 0;

// Mutex 벤치마크
static void BM_Mutex_Write(benchmark::State& state) {
    for (auto _ : state) {
        std::lock_guard<std::mutex> lock(mtx);
        shared_data++;
    }
}
BENCHMARK(BM_Mutex_Write)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

// Shared Mutex 읽기 벤치마크
static void BM_SharedMutex_Read(benchmark::State& state) {
    for (auto _ : state) {
        std::shared_lock<std::shared_mutex> lock(sh_mtx);
        benchmark::DoNotOptimize(shared_data);
    }
}
BENCHMARK(BM_SharedMutex_Read)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

// Shared Mutex 쓰기 벤치마크
static void BM_SharedMutex_Write(benchmark::State& state) {
    for (auto _ : state) {
        std::unique_lock<std::shared_mutex> lock(sh_mtx);
        shared_data++;
    }
}
BENCHMARK(BM_SharedMutex_Write)->Threads(1)->Threads(2)->Threads(4)->Threads(8);

BENCHMARK_MAIN();
```

**실행 결과:**
```
Benchmark                           Time        CPU   Iterations
-----------------------------------------------------------------
BM_Mutex_Write/threads:1           25 ns      25 ns   28000000
BM_Mutex_Write/threads:4          125 ns     500 ns    5600000
BM_Mutex_Write/threads:8          250 ns    2000 ns    2800000

BM_SharedMutex_Read/threads:1      15 ns      15 ns   46000000
BM_SharedMutex_Read/threads:4      18 ns      72 ns   38800000
BM_SharedMutex_Read/threads:8      20 ns     160 ns   35000000

BM_SharedMutex_Write/threads:1     30 ns      30 ns   23000000
BM_SharedMutex_Write/threads:4    150 ns     600 ns    4666667
BM_SharedMutex_Write/threads:8    300 ns    2400 ns    2333333
```

**분석:**
- 읽기 위주: Shared Mutex가 유리
- 쓰기 비율 높음: 일반 Mutex와 비슷

---

#### Go Benchmark

```go
package main

import (
    "sync"
    "testing"
)

var (
    counter   int
    mtx       sync.Mutex
    rwMtx     sync.RWMutex
)

// Mutex 벤치마크
func BenchmarkMutex(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mtx.Lock()
            counter++
            mtx.Unlock()
        }
    })
}

// RWMutex 읽기
func BenchmarkRWMutex_Read(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            rwMtx.RLock()
            _ = counter
            rwMtx.RUnlock()
        }
    })
}

// RWMutex 쓰기
func BenchmarkRWMutex_Write(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            rwMtx.Lock()
            counter++
            rwMtx.Unlock()
        }
    })
}

// 실행: go test -bench=. -cpu=1,2,4,8
```

---

### 2. 프로파일링

#### CPU 프로파일링 (Go)

```go
package main

import (
    "os"
    "runtime/pprof"
)

func main() {
    // CPU 프로파일 시작
    f, _ := os.Create("cpu.prof")
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    // 프로그램 실행
    runProgram()
}

// 분석
// go tool pprof cpu.prof
// (pprof) top10
// (pprof) list functionName
```

#### perf (Linux)

```bash
# 프로그램 실행하며 프로파일링
perf record -g ./program

# 결과 분석
perf report

# CPU 캐시 미스 측정
perf stat -e cache-misses,cache-references ./program

# Lock contention 측정
perf record -e syscalls:sys_enter_futex ./program
```

---

## 🚀 최적화 기법

### 1. False Sharing 제거

#### 문제

```cpp
// ❌ False Sharing 발생
struct Counter {
    std::atomic<int> count1;  // Cache line 1
    std::atomic<int> count2;  // Cache line 1 (같은 라인!)
};

Counter counters;

// Thread 1
counters.count1.fetch_add(1);  // Cache line invalidation

// Thread 2
counters.count2.fetch_add(1);  // Cache line invalidation
// 서로 영향을 줌!
```

#### 해결책

```cpp
// ✅ Padding으로 분리
struct alignas(64) Counter {
    std::atomic<int> count;
    char padding[64 - sizeof(std::atomic<int>)];
};

Counter counters[2];

// Thread 1
counters[0].count.fetch_add(1);  // Cache line 1

// Thread 2
counters[1].count.fetch_add(1);  // Cache line 2
// 독립적!
```

#### C++17 hardware_destructive_interference_size

```cpp
#include <new>

struct alignas(std::hardware_destructive_interference_size) Counter {
    std::atomic<int> count;
};
```

---

### 2. Lock Contention 감소

#### 문제: 단일 락

```cpp
// ❌ 모든 스레드가 하나의 락 경쟁
std::mutex global_mtx;
std::unordered_map<int, int> global_map;

void insert(int key, int value) {
    std::lock_guard<std::mutex> lock(global_mtx);
    global_map[key] = value;
}
```

#### 해결책 1: Lock Striping

```cpp
// ✅ 여러 락으로 분산
template <typename K, typename V>
class StripedMap {
    static constexpr size_t NUM_STRIPES = 16;

    struct Stripe {
        std::mutex mtx;
        std::unordered_map<K, V> map;
    };

    Stripe stripes_[NUM_STRIPES];

    Stripe& get_stripe(K key) {
        size_t hash = std::hash<K>{}(key);
        return stripes_[hash % NUM_STRIPES];
    }

public:
    void insert(K key, V value) {
        auto& stripe = get_stripe(key);
        std::lock_guard<std::mutex> lock(stripe.mtx);
        stripe.map[key] = value;
    }

    bool find(K key, V& value) {
        auto& stripe = get_stripe(key);
        std::lock_guard<std::mutex> lock(stripe.mtx);
        auto it = stripe.map.find(key);
        if (it != stripe.map.end()) {
            value = it->second;
            return true;
        }
        return false;
    }
};
```

#### 해결책 2: Lock-Free 자료구조

```cpp
// ✅ Lock 없이
#include <folly/concurrency/ConcurrentHashMap.h>

folly::ConcurrentHashMap<int, int> map;

void insert(int key, int value) {
    map.insert(key, value);  // Lock-free!
}
```

---

### 3. Read-Write 비율 최적화

#### Reader가 많은 경우

```cpp
// ✅ Shared Mutex 사용
std::shared_mutex sh_mtx;
std::map<int, int> data;

// 읽기 (여러 스레드 동시 가능)
int read(int key) {
    std::shared_lock<std::shared_mutex> lock(sh_mtx);
    return data[key];
}

// 쓰기 (독점)
void write(int key, int value) {
    std::unique_lock<std::shared_mutex> lock(sh_mtx);
    data[key] = value;
}
```

#### RCU (Read-Copy-Update) 패턴

```cpp
template <typename T>
class RCUData {
    std::atomic<T*> data_;
    std::mutex write_mtx_;

public:
    RCUData(T* initial) : data_(initial) {}

    // 읽기 (Lock-free!)
    T* read() {
        return data_.load(std::memory_order_acquire);
    }

    // 쓰기 (Copy-on-write)
    void write(T* new_data) {
        std::lock_guard<std::mutex> lock(write_mtx_);

        T* old_data = data_.load(std::memory_order_acquire);

        // 원자적으로 교체
        data_.store(new_data, std::memory_order_release);

        // 이전 데이터는 나중에 삭제 (grace period)
        // 실제로는 Hazard Pointer 등 사용
        delete old_data;
    }
};
```

---

### 4. Memory Ordering 최적화

#### Sequential Consistency (가장 강함, 느림)

```cpp
std::atomic<int> x{0}, y{0};

// Thread 1
x.store(1, std::memory_order_seq_cst);

// Thread 2
y.store(1, std::memory_order_seq_cst);
```

#### Acquire-Release (균형)

```cpp
std::atomic<bool> ready{false};
int data = 0;

// Producer
data = 42;
ready.store(true, std::memory_order_release);

// Consumer
while (!ready.load(std::memory_order_acquire)) {}
assert(data == 42);  // 보장됨
```

#### Relaxed (가장 약함, 빠름)

```cpp
std::atomic<int> counter{0};

// 단순 카운터 (순서 무관)
counter.fetch_add(1, std::memory_order_relaxed);
```

**성능 비교:**
```
Sequential Consistency:  ~20 ns
Acquire-Release:         ~5 ns
Relaxed:                 ~2 ns
```

---

### 5. Work Stealing

#### 문제: Load Imbalance

```cpp
// ❌ 일부 스레드는 바쁘고 일부는 놈
ThreadPool pool(4);

// Thread 1: 긴 작업
pool.submit([]() { heavy_task(); });

// Thread 2-4: 짧은 작업
pool.submit([]() { light_task(); });
```

#### 해결책: Work Stealing Queue

```cpp
class WorkStealingThreadPool {
    struct WorkerThread {
        std::deque<Task> local_queue;
        std::mutex mtx;

        void push_back(Task task) {
            std::lock_guard<std::mutex> lock(mtx);
            local_queue.push_back(std::move(task));
        }

        bool pop_front(Task& task) {
            std::lock_guard<std::mutex> lock(mtx);
            if (local_queue.empty()) return false;
            task = std::move(local_queue.front());
            local_queue.pop_front();
            return true;
        }

        // 다른 스레드가 뒤에서 훔침
        bool steal(Task& task) {
            std::lock_guard<std::mutex> lock(mtx);
            if (local_queue.empty()) return false;
            task = std::move(local_queue.back());
            local_queue.pop_back();
            return true;
        }
    };

    std::vector<WorkerThread> workers_;

    void worker_loop(int worker_id) {
        while (!stop_) {
            Task task;

            // 1. 자신의 큐에서 가져오기
            if (workers_[worker_id].pop_front(task)) {
                task();
                continue;
            }

            // 2. 다른 스레드에서 훔치기
            bool stolen = false;
            for (size_t i = 0; i < workers_.size(); ++i) {
                if (i != worker_id && workers_[i].steal(task)) {
                    task();
                    stolen = true;
                    break;
                }
            }

            if (!stolen) {
                std::this_thread::yield();
            }
        }
    }
};
```

---

### 6. Batching

#### 문제: 빈번한 락 획득

```cpp
// ❌ 매번 락 획득
for (int i = 0; i < 1000; ++i) {
    std::lock_guard<std::mutex> lock(mtx);
    queue.push(i);  // 1000번 락!
}
```

#### 해결책: 배치 처리

```cpp
// ✅ 배치로 묶기
std::vector<int> batch;
batch.reserve(1000);

for (int i = 0; i < 1000; ++i) {
    batch.push_back(i);
}

{
    std::lock_guard<std::mutex> lock(mtx);
    for (int item : batch) {
        queue.push(item);  // 1번 락!
    }
}
```

---

### 7. Thread Pool 크기 조정

#### CPU-Bound 작업

```cpp
// 물리 코어 수 = 최적
int num_threads = std::thread::hardware_concurrency();
ThreadPool pool(num_threads);
```

#### I/O-Bound 작업

```cpp
// 코어 수보다 많이
int num_threads = std::thread::hardware_concurrency() * 2;
ThreadPool pool(num_threads);
```

#### 동적 조정

```go
type AdaptivePool struct {
    minWorkers int
    maxWorkers int
    current    int
    tasks      chan Task
}

func (p *AdaptivePool) Adjust() {
    queueSize := len(p.tasks)

    if queueSize > p.current*10 && p.current < p.maxWorkers {
        // 큐가 넘침: 워커 추가
        p.addWorker()
    } else if queueSize < p.current*2 && p.current > p.minWorkers {
        // 큐가 비어있음: 워커 감소
        p.removeWorker()
    }
}
```

---

## 📈 확장성 분석

### Amdahl's Law

```
가속비 = 1 / ((1 - P) + P/N)

P: 병렬화 가능한 비율
N: 프로세서 수
```

**예시:**
```
P = 90% (병렬화 가능)
N = 4 (코어)

가속비 = 1 / ((1 - 0.9) + 0.9/4)
      = 1 / (0.1 + 0.225)
      = 3.08배
```

**결론:** 순차 부분(10%)이 병목

---

### 확장성 측정

```cpp
#include <benchmark/benchmark.h>

static void BM_ParallelTask(benchmark::State& state) {
    const int num_threads = state.range(0);

    for (auto _ : state) {
        std::vector<std::thread> threads;

        for (int i = 0; i < num_threads; ++i) {
            threads.emplace_back([]() {
                compute();
            });
        }

        for (auto& t : threads) {
            t.join();
        }
    }
}

BENCHMARK(BM_ParallelTask)
    ->Arg(1)
    ->Arg(2)
    ->Arg(4)
    ->Arg(8)
    ->Arg(16);

// 결과 분석:
// 1 thread:  100ms (baseline)
// 2 threads:  55ms (1.8x speedup)
// 4 threads:  30ms (3.3x speedup)
// 8 threads:  20ms (5.0x speedup) <- 이상적: 8x
// 확장성 한계 확인
```

---

## 🎯 실전 사례

### 사례 1: 이미지 처리 최적화

#### Before (느림)

```cpp
void processImages(const std::vector<Image>& images) {
    std::mutex results_mtx;
    std::vector<ProcessedImage> results;

    std::vector<std::thread> threads;

    for (const auto& img : images) {
        threads.emplace_back([&]() {
            auto processed = processImage(img);  // CPU-bound

            std::lock_guard<std::mutex> lock(results_mtx);
            results.push_back(processed);  // Lock contention!
        });
    }

    for (auto& t : threads) {
        t.join();
    }
}
```

#### After (빠름)

```cpp
void processImages(const std::vector<Image>& images) {
    const int num_threads = std::thread::hardware_concurrency();
    std::vector<std::vector<ProcessedImage>> thread_results(num_threads);

    std::vector<std::thread> threads;

    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back([&, i]() {
            // 각 스레드는 자신의 범위 처리 (Lock 없음!)
            for (size_t j = i; j < images.size(); j += num_threads) {
                thread_results[i].push_back(processImage(images[j]));
            }
        });
    }

    for (auto& t : threads) {
        t.join();
    }

    // 결과 병합 (한 번만)
    std::vector<ProcessedImage> results;
    for (const auto& thread_result : thread_results) {
        results.insert(results.end(), thread_result.begin(), thread_result.end());
    }
}

// 성능: 3배 향상!
```

---

### 사례 2: 로그 시스템 최적화

#### Before

```cpp
// ❌ 락 경합 심함
class Logger {
    std::mutex mtx_;
    std::ofstream file_;

public:
    void log(const std::string& msg) {
        std::lock_guard<std::mutex> lock(mtx_);
        file_ << msg << std::endl;  // I/O blocking!
    }
};

// 많은 스레드에서 호출
logger.log("Message");
```

#### After

```cpp
// ✅ 비동기 로깅
class AsyncLogger {
    folly::MPMCQueue<std::string> queue_{10000};
    std::thread worker_;
    std::atomic<bool> stop_{false};

    void worker_loop() {
        std::ofstream file_("log.txt");
        std::string msg;

        while (!stop_.load()) {
            if (queue_.read(msg)) {
                file_ << msg << std::endl;
            } else {
                std::this_thread::sleep_for(std::chrono::milliseconds(1));
            }
        }
    }

public:
    AsyncLogger() {
        worker_ = std::thread([this]() { worker_loop(); });
    }

    ~AsyncLogger() {
        stop_.store(true);
        worker_.join();
    }

    void log(std::string msg) {
        queue_.blockingWrite(std::move(msg));  // 빠름!
    }
};

// 성능: 10배 향상!
```

---

## 📚 체크리스트

### 최적화 전

- [ ] 프로파일링 완료
- [ ] 병목 지점 식별
- [ ] 현재 성능 측정 완료
- [ ] 목표 성능 설정

### 최적화 중

- [ ] 한 번에 하나씩 변경
- [ ] 변경 후 벤치마크
- [ ] 회귀 테스트
- [ ] 코드 리뷰

### 최적화 후

- [ ] 목표 달성 확인
- [ ] 문서화
- [ ] 모니터링 설정
- [ ] 유지보수 계획

---

## ⚠️ 주의사항

1. **조기 최적화 금지**: 필요할 때만
2. **가독성 vs 성능**: 균형 유지
3. **플랫폼 의존성**: 다른 환경에서 테스트
4. **유지보수성**: 복잡도 증가 주의

---

## 🔗 추가 리소스

- Google Benchmark: https://github.com/google/benchmark
- perf tutorial: https://perf.wiki.kernel.org/
- Intel VTune: https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html

---

*다음: [참고 자료](./references.md)*
