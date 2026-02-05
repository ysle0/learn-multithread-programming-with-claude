# Fan-Out/Fan-In 패턴

## 개요

Fan-Out/Fan-In 패턴은 작업을 여러 병렬 worker에 분배하고(fan-out), 그 결과를 집계합니다(fan-in). 이 패턴은 데이터 병렬성을 활용하여 독립적인 작업 청크를 동시에 처리한 후, 결과를 최종 출력으로 결합합니다. 확장성을 달성하기 위한 가장 효과적인 패턴 중 하나입니다.

## 문제 정의

많은 계산 문제는 독립적인 하위 작업으로 나눌 수 있습니다:
- 대규모 데이터셋 처리 (map-reduce)
- 여러 소스에 대한 병렬 검색
- 분산 계산
- 독립적인 항목의 배치 처리

순차적으로 처리하면 사용 가능한 병렬성을 낭비하게 되고 시간이 너무 오래 걸립니다.

## 솔루션 아키텍처

```
                    ┌──────────────────┐
                    │   Input Data     │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  FAN-OUT         │
                    │  (Distribute)    │
                    └──┬───┬───┬───┬───┘
                       │   │   │   │
            ┌──────────┘   │   │   └──────────┐
            │              │   │              │
       ┌────▼────┐    ┌───▼───┐    ┌────▼────┐
       │ Worker  │    │Worker │    │ Worker  │
       │    1    │    │   2   │    │    N    │
       └────┬────┘    └───┬───┘    └────┬────┘
            │             │              │
            └─────────┐   │   ┌──────────┘
                      │   │   │
                 ┌────▼───▼───▼────┐
                 │    FAN-IN        │
                 │   (Aggregate)    │
                 └────────┬─────────┘
                          │
                 ┌────────▼─────────┐
                 │  Combined Result │
                 └──────────────────┘

특징:
- Worker가 데이터를 병렬로 처리 (fan-out)
- Worker 간 의존성 없음
- 집계기가 결과를 결합 (fan-in)
- 확장성: worker 추가 가능
```

## 기본 구현

### 간단한 Fan-Out/Fan-In

```cpp
#include <vector>
#include <thread>
#include <future>
#include <numeric>
#include <algorithm>

// future를 사용한 Fan-out/Fan-in
template<typename InputIterator, typename Function>
auto parallel_transform(InputIterator begin, InputIterator end,
                       Function func, size_t num_workers)
    -> std::vector<decltype(func(*begin))> {

    using ResultType = decltype(func(*begin));

    size_t total_items = std::distance(begin, end);
    size_t chunk_size = (total_items + num_workers - 1) / num_workers;

    std::vector<std::future<std::vector<ResultType>>> futures;

    // Fan-out: worker 실행
    auto chunk_begin = begin;
    for (size_t i = 0; i < num_workers && chunk_begin != end; ++i) {
        auto chunk_end = chunk_begin;
        std::advance(chunk_end, std::min(chunk_size,
            static_cast<size_t>(std::distance(chunk_begin, end))));

        futures.push_back(std::async(std::launch::async,
            [chunk_begin, chunk_end, func]() {
                std::vector<ResultType> results;
                for (auto it = chunk_begin; it != chunk_end; ++it) {
                    results.push_back(func(*it));
                }
                return results;
            }
        ));

        chunk_begin = chunk_end;
    }

    // Fan-in: 결과 수집
    std::vector<ResultType> combined_results;
    for (auto& future : futures) {
        auto partial_results = future.get();
        combined_results.insert(combined_results.end(),
                               partial_results.begin(),
                               partial_results.end());
    }

    return combined_results;
}

// 사용 예제
int main() {
    std::vector<int> data(1000);
    std::iota(data.begin(), data.end(), 1);  // 1, 2, 3, ..., 1000

    // 각 숫자를 병렬로 제곱
    auto results = parallel_transform(data.begin(), data.end(),
        [](int x) { return x * x; },
        std::thread::hardware_concurrency()
    );

    std::cout << "Processed " << results.size() << " items\n";
    return 0;
}
```

## Map-Reduce 패턴

```cpp
template<typename InputIterator, typename MapFunc, typename ReduceFunc>
auto map_reduce(InputIterator begin, InputIterator end,
               MapFunc mapper, ReduceFunc reducer,
               size_t num_workers) {

    using MappedType = decltype(mapper(*begin));

    size_t total_items = std::distance(begin, end);
    size_t chunk_size = (total_items + num_workers - 1) / num_workers;

    std::vector<std::future<std::vector<MappedType>>> futures;

    // Map 단계 (fan-out)
    auto chunk_begin = begin;
    for (size_t i = 0; i < num_workers && chunk_begin != end; ++i) {
        auto chunk_end = chunk_begin;
        std::advance(chunk_end, std::min(chunk_size,
            static_cast<size_t>(std::distance(chunk_begin, end))));

        futures.push_back(std::async(std::launch::async,
            [chunk_begin, chunk_end, mapper]() {
                std::vector<MappedType> results;
                for (auto it = chunk_begin; it != chunk_end; ++it) {
                    results.push_back(mapper(*it));
                }
                return results;
            }
        ));

        chunk_begin = chunk_end;
    }

    // Reduce 단계 (fan-in)
    std::vector<MappedType> all_mapped;
    for (auto& future : futures) {
        auto partial = future.get();
        all_mapped.insert(all_mapped.end(), partial.begin(), partial.end());
    }

    // 최종 리듀스
    if (all_mapped.empty()) {
        throw std::runtime_error("No data to reduce");
    }

    auto result = all_mapped[0];
    for (size_t i = 1; i < all_mapped.size(); ++i) {
        result = reducer(result, all_mapped[i]);
    }

    return result;
}

// 예제: 병렬 합산
int main() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 1);

    auto sum = map_reduce(
        data.begin(), data.end(),
        [](int x) { return x; },           // Map: 항등 함수
        [](int a, int b) { return a + b; }, // Reduce: 합산
        std::thread::hardware_concurrency()
    );

    std::cout << "Sum: " << sum << "\n";
    return 0;
}
```

## Fan-Out/Fan-In을 사용한 Worker Pool

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>

template<typename Input, typename Output>
class WorkerPool {
private:
    struct Task {
        Input data;
        size_t task_id;
    };

    std::vector<std::thread> workers_;
    std::queue<Task> task_queue_;
    std::vector<Output> results_;

    std::mutex queue_mutex_;
    std::mutex results_mutex_;
    std::condition_variable task_cv_;

    std::atomic<bool> stopped_{false};
    std::atomic<size_t> active_workers_{0};
    std::atomic<size_t> completed_tasks_{0};

    std::function<Output(Input)> processor_;

public:
    WorkerPool(size_t num_workers, std::function<Output(Input)> processor)
        : processor_(std::move(processor)) {

        for (size_t i = 0; i < num_workers; ++i) {
            workers_.emplace_back([this] { worker_thread(); });
        }
    }

    ~WorkerPool() {
        stop();
    }

    void submit(const std::vector<Input>& inputs) {
        results_.resize(inputs.size());

        {
            std::lock_guard<std::mutex> lock(queue_mutex_);
            for (size_t i = 0; i < inputs.size(); ++i) {
                task_queue_.push(Task{inputs[i], i});
            }
        }

        task_cv_.notify_all();
    }

    std::vector<Output> wait_all() {
        // 모든 작업이 완료될 때까지 대기
        while (completed_tasks_.load() < results_.size()) {
            std::this_thread::sleep_for(std::chrono::milliseconds(10));
        }

        return results_;
    }

    void stop() {
        stopped_.store(true);
        task_cv_.notify_all();

        for (auto& worker : workers_) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }

private:
    void worker_thread() {
        while (!stopped_.load()) {
            Task task;
            bool has_task = false;

            {
                std::unique_lock<std::mutex> lock(queue_mutex_);
                task_cv_.wait_for(lock, std::chrono::milliseconds(100),
                    [this] { return !task_queue_.empty() || stopped_.load(); });

                if (!task_queue_.empty()) {
                    task = task_queue_.front();
                    task_queue_.pop();
                    has_task = true;
                }
            }

            if (has_task) {
                active_workers_.fetch_add(1);

                try {
                    Output result = processor_(task.data);

                    {
                        std::lock_guard<std::mutex> lock(results_mutex_);
                        results_[task.task_id] = std::move(result);
                    }

                    completed_tasks_.fetch_add(1);
                } catch (const std::exception& e) {
                    std::cerr << "Worker 오류: " << e.what() << "\n";
                }

                active_workers_.fetch_sub(1);
            }
        }
    }
};

// 사용법
int main() {
    WorkerPool<int, int> pool(4, [](int x) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        return x * x;
    });

    std::vector<int> inputs = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    pool.submit(inputs);
    auto results = pool.wait_all();

    for (size_t i = 0; i < results.size(); ++i) {
        std::cout << inputs[i] << "^2 = " << results[i] << "\n";
    }

    return 0;
}
```

## 고급: 계층적 Fan-Out/Fan-In

```cpp
// 더 나은 확장성을 위한 트리 기반 리듀스
template<typename T, typename ReduceFunc>
T hierarchical_reduce(const std::vector<T>& data, ReduceFunc reducer,
                      size_t num_workers) {
    if (data.empty()) {
        throw std::runtime_error("Cannot reduce empty data");
    }

    if (data.size() == 1) {
        return data[0];
    }

    // 레벨 1: 부분 리듀스를 위해 worker에 fan-out
    size_t chunk_size = (data.size() + num_workers - 1) / num_workers;
    std::vector<std::future<T>> level1_futures;

    for (size_t i = 0; i < data.size(); i += chunk_size) {
        size_t end = std::min(i + chunk_size, data.size());

        level1_futures.push_back(std::async(std::launch::async,
            [&data, i, end, reducer]() {
                T result = data[i];
                for (size_t j = i + 1; j < end; ++j) {
                    result = reducer(result, data[j]);
                }
                return result;
            }
        ));
    }

    // 레벨 2: 부분 결과 수집
    std::vector<T> partial_results;
    for (auto& future : level1_futures) {
        partial_results.push_back(future.get());
    }

    // 레벨 3: 최종 리듀스 (더 많은 레벨을 위해 재귀 가능)
    T final_result = partial_results[0];
    for (size_t i = 1; i < partial_results.size(); ++i) {
        final_result = reducer(final_result, partial_results[i]);
    }

    return final_result;
}

// 예제: 병렬로 최댓값 찾기
int main() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 1);
    std::random_shuffle(data.begin(), data.end());

    auto max_value = hierarchical_reduce(data,
        [](int a, int b) { return std::max(a, b); },
        std::thread::hardware_concurrency()
    );

    std::cout << "Maximum: " << max_value << "\n";
    return 0;
}
```

## Scatter-Gather 패턴

```cpp
#include <map>

template<typename Key, typename Value>
class ScatterGather {
public:
    using DataSource = std::function<Value(Key)>;

    ScatterGather(const std::vector<DataSource>& sources)
        : sources_(sources) {}

    // 모든 소스에 요청을 분산하고 결과를 수집
    std::map<size_t, Value> query(const Key& key) {
        std::vector<std::future<Value>> futures;

        // Scatter: 모든 소스에 병렬로 쿼리
        for (size_t i = 0; i < sources_.size(); ++i) {
            futures.push_back(std::async(std::launch::async,
                sources_[i], key
            ));
        }

        // Gather: 모든 결과 수집
        std::map<size_t, Value> results;
        for (size_t i = 0; i < futures.size(); ++i) {
            try {
                results[i] = futures[i].get();
            } catch (const std::exception& e) {
                std::cerr << "소스 " << i << " 실패: "
                          << e.what() << "\n";
            }
        }

        return results;
    }

    // 타임아웃이 있는 쿼리: 부분 결과 반환
    template<typename Rep, typename Period>
    std::map<size_t, Value> query_with_timeout(
        const Key& key,
        const std::chrono::duration<Rep, Period>& timeout) {

        std::vector<std::future<Value>> futures;

        for (size_t i = 0; i < sources_.size(); ++i) {
            futures.push_back(std::async(std::launch::async,
                sources_[i], key
            ));
        }

        std::map<size_t, Value> results;
        auto deadline = std::chrono::steady_clock::now() + timeout;

        for (size_t i = 0; i < futures.size(); ++i) {
            auto remaining = deadline - std::chrono::steady_clock::now();

            if (remaining <= std::chrono::seconds(0)) {
                break;  // 타임아웃
            }

            if (futures[i].wait_for(remaining) == std::future_status::ready) {
                try {
                    results[i] = futures[i].get();
                } catch (const std::exception& e) {
                    std::cerr << "소스 " << i << " 오류: "
                              << e.what() << "\n";
                }
            }
        }

        return results;
    }

private:
    std::vector<DataSource> sources_;
};

// 예제: 분산 검색
struct SearchResult {
    std::vector<std::string> items;
};

int main() {
    // 시뮬레이션된 데이터 소스
    std::vector<ScatterGather<std::string, SearchResult>::DataSource> sources = {
        [](const std::string& query) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            return SearchResult{{"result1_source0", "result2_source0"}};
        },
        [](const std::string& query) {
            std::this_thread::sleep_for(std::chrono::milliseconds(150));
            return SearchResult{{"result1_source1"}};
        },
        [](const std::string& query) {
            std::this_thread::sleep_for(std::chrono::milliseconds(200));
            return SearchResult{{"result1_source2", "result2_source2", "result3_source2"}};
        }
    };

    ScatterGather<std::string, SearchResult> sg(sources);

    // 250ms 타임아웃으로 모든 소스에 쿼리
    auto results = sg.query_with_timeout("search term",
                                         std::chrono::milliseconds(250));

    std::cout << "Got results from " << results.size() << " sources:\n";
    for (const auto& [source_id, result] : results) {
        std::cout << "  Source " << source_id << ": "
                  << result.items.size() << " items\n";
    }

    return 0;
}
```

## 부하 분산 전략

### Round-Robin 분배

```cpp
template<typename Task>
class RoundRobinDistributor {
private:
    std::vector<std::queue<Task>> worker_queues_;
    std::vector<std::mutex> mutexes_;
    size_t next_worker_ = 0;

public:
    explicit RoundRobinDistributor(size_t num_workers)
        : worker_queues_(num_workers), mutexes_(num_workers) {}

    void distribute(Task task) {
        size_t worker = next_worker_;
        next_worker_ = (next_worker_ + 1) % worker_queues_.size();

        std::lock_guard<std::mutex> lock(mutexes_[worker]);
        worker_queues_[worker].push(std::move(task));
    }
};
```

### Work-Stealing 분배

```cpp
template<typename Task>
class WorkStealingDistributor {
private:
    std::vector<std::deque<Task>> worker_queues_;
    std::vector<std::mutex> mutexes_;

public:
    explicit WorkStealingDistributor(size_t num_workers)
        : worker_queues_(num_workers), mutexes_(num_workers) {}

    // Worker가 자신의 큐에 추가
    void push_local(size_t worker_id, Task task) {
        std::lock_guard<std::mutex> lock(mutexes_[worker_id]);
        worker_queues_[worker_id].push_front(std::move(task));
    }

    // Worker가 유휴 상태일 때 다른 큐에서 작업을 훔침
    std::optional<Task> steal(size_t worker_id) {
        // 다른 worker에서 훔치기 시도
        for (size_t i = 1; i < worker_queues_.size(); ++i) {
            size_t victim = (worker_id + i) % worker_queues_.size();

            std::lock_guard<std::mutex> lock(mutexes_[victim]);
            if (!worker_queues_[victim].empty()) {
                Task task = std::move(worker_queues_[victim].back());
                worker_queues_[victim].pop_back();
                return task;
            }
        }

        return std::nullopt;  // 훔칠 작업 없음
    }
};
```

## 집계 전략

### 합산 집계

```cpp
template<typename T>
T sum_aggregator(const std::vector<T>& results) {
    return std::accumulate(results.begin(), results.end(), T{});
}
```

### 최솟값/최댓값 집계

```cpp
template<typename T>
T max_aggregator(const std::vector<T>& results) {
    return *std::max_element(results.begin(), results.end());
}
```

### 사용자 정의 집계

```cpp
template<typename T>
struct AggregatedResult {
    T min_value;
    T max_value;
    double average;
    size_t count;
};

template<typename T>
AggregatedResult<T> stats_aggregator(const std::vector<T>& results) {
    if (results.empty()) {
        throw std::runtime_error("Cannot aggregate empty results");
    }

    AggregatedResult<T> agg;
    agg.min_value = *std::min_element(results.begin(), results.end());
    agg.max_value = *std::max_element(results.begin(), results.end());
    agg.count = results.size();

    T sum = std::accumulate(results.begin(), results.end(), T{});
    agg.average = static_cast<double>(sum) / results.size();

    return agg;
}
```

## 실제 응용 사례

### 1. 병렬 이미지 처리

```cpp
// Fan-out: 이미지 타일을 병렬로 처리
// Fan-in: 타일을 다시 결합
std::vector<ImageTile> tiles = split_image(large_image);

auto processed_tiles = parallel_transform(tiles.begin(), tiles.end(),
    [](const ImageTile& tile) {
        return apply_filter(tile);
    },
    num_workers
);

Image result = stitch_tiles(processed_tiles);
```

### 2. 분산 검색

```
Query ───┬──▶ Database 1 ──┐
         ├──▶ Database 2 ──┤
         ├──▶ Database 3 ──┼──▶ Merge Results
         └──▶ Database N ──┘
```

### 3. 웹 크롤링

```cpp
// Fan-out: 여러 URL을 병렬로 가져오기
// Fan-in: 추출된 데이터 집계
std::vector<std::string> urls = get_urls_to_crawl();

auto pages = parallel_transform(urls.begin(), urls.end(),
    [](const std::string& url) {
        return fetch_and_parse(url);
    },
    100  // 100개 동시 가져오기
);

auto all_links = merge_links(pages);
```

### 4. 배치 데이터 처리

```cpp
// 대규모 데이터셋을 병렬 청크로 처리
std::vector<DataChunk> chunks = partition_data(huge_dataset, num_workers);

auto results = parallel_transform(chunks.begin(), chunks.end(),
    [](const DataChunk& chunk) {
        return process_chunk(chunk);
    },
    num_workers
);

auto final_result = reduce_results(results);
```

### 5. 금융 포트폴리오 분석

```cpp
// 여러 주식을 병렬로 분석
std::vector<Stock> portfolio = get_portfolio();

auto analyses = parallel_transform(portfolio.begin(), portfolio.end(),
    [](const Stock& stock) {
        return calculate_risk_metrics(stock);
    },
    num_workers
);

PortfolioRisk total_risk = aggregate_risk(analyses);
```

## 성능 고려 사항

### 암달의 법칙 (Amdahl's Law)

```
Speedup = 1 / ((1 - P) + P/N)

여기서:
  P = 병렬화 가능 비율 (0에서 1)
  N = Worker 수

예제:
  90% 병렬화 가능 (P=0.9), worker 10개:
  Speedup = 1 / (0.1 + 0.9/10) = 5.26배

  완전 병렬화 (P=1.0), worker 10개:
  Speedup = 10배
```

### 오버헤드 분석

```cpp
// 오버헤드 측정
auto start = std::chrono::high_resolution_clock::now();

// 순차 처리
int sum_seq = std::accumulate(data.begin(), data.end(), 0);

auto seq_time = std::chrono::high_resolution_clock::now() - start;

// 병렬 처리
start = std::chrono::high_resolution_clock::now();

auto sum_par = map_reduce(data.begin(), data.end(),
    [](int x) { return x; },
    [](int a, int b) { return a + b; },
    num_workers
);

auto par_time = std::chrono::high_resolution_clock::now() - start;

std::cout << "Sequential: " << seq_time.count() << "ns\n";
std::cout << "Parallel: " << par_time.count() << "ns\n";
std::cout << "Speedup: " << (seq_time / par_time) << "x\n";
```

### 최적 Worker 수

```cpp
// CPU 바운드 작업의 경우
size_t optimal_workers = std::thread::hardware_concurrency();

// I/O 바운드 작업의 경우 (훨씬 더 높을 수 있음)
size_t optimal_workers = std::thread::hardware_concurrency() * 10;

// 혼합 워크로드의 경우 (벤치마크로 최적값 탐색)
for (size_t workers = 1; workers <= 32; workers *= 2) {
    auto time = benchmark_with_workers(workers);
    std::cout << "Workers: " << workers << ", Time: " << time << "\n";
}
```

## 흔한 실수

### 1. 작은 작업에서 오버헤드가 지배적

```cpp
// 나쁜 예: 병렬 오버헤드 > 작업량
std::vector<int> small_data = {1, 2, 3};
auto result = parallel_sum(small_data);  // 순차보다 느림!

// 좋은 예: 충분히 큰 워크로드만 병렬화
if (data.size() > 1000) {
    result = parallel_sum(data);
} else {
    result = sequential_sum(data);
}
```

### 2. 부하 불균형

```cpp
// 나쁜 예: 불균등한 작업 분배
// Worker 1: 1000개 항목
// Worker 2: 10개 항목
// Worker 2가 일찍 끝나고 유휴 상태

// 좋은 예: 워크로드 균형 맞추기 또는 work stealing 사용
```

### 3. 메모리 경합

```cpp
// 나쁜 예: 모든 worker가 같은 위치에 쓰기
std::atomic<int> counter{0};
parallel_for_each([&](int x) {
    counter++;  // 경합!
});

// 좋은 예: worker별 누적 후 집계
std::vector<int> per_worker_counts(num_workers, 0);
// 각 worker가 자신의 카운터를 업데이트
// 최종 fan-in에서 모든 카운터를 합산
```

## 테스트 전략

### 정확성

```cpp
// 병렬 결과가 순차 결과와 일치하는지 확인
auto seq_result = sequential_process(data);
auto par_result = parallel_process(data);
assert(seq_result == par_result);
```

### 확장성

```cpp
// Worker 수를 늘려가며 속도 향상 측정
for (size_t workers = 1; workers <= 16; workers *= 2) {
    auto time = benchmark(data, workers);
    double speedup = baseline_time / time;
    std::cout << "Workers: " << workers
              << ", Speedup: " << speedup << "x\n";
}
```

## 장단점

### 장점
- 데이터 병렬 문제에 대한 뛰어난 확장성
- 간단한 멘탈 모델
- future/async로 쉽게 구현 가능
- 당혹적 병렬(embarrassingly parallel) 문제에서 거의 선형적인 속도 향상
- 우수한 CPU 활용률

### 단점
- 작은 작업에 대한 오버헤드
- 잠재적 부하 불균형
- 데이터 복제를 위한 메모리 오버헤드
- 작업 간 의존성이 있는 경우 부적합
- 집계가 병목이 될 수 있음

## 모범 사례

1. **작업이 독립적인지 확인** (공유 상태 없음)
2. **작업 크기를 적절하게 설정** (오버헤드와 병렬성의 균형)
3. **worker 간 워크로드 균형 유지**
4. **적절한 수의 worker 사용** (프로파일링으로 최적값 찾기)
5. **worker에서의 오류 처리** (하나의 실패가 전체를 중단시키지 않도록)
6. **대규모 결과 집합에는 계층적 fan-in 고려**
7. **worker 활용률 모니터링**으로 부하 불균형 감지

## 요약

Fan-Out/Fan-In은 다음에 이상적입니다:
- 데이터 병렬 문제
- 당혹적 병렬(embarrassingly parallel) 워크로드
- 배치 처리
- 분산 계산

적합한 문제에 적용할 때 거의 선형적인 속도 향상을 제공하는, 병렬성을 달성하기 위한 가장 간단하고 효과적인 패턴 중 하나입니다.
