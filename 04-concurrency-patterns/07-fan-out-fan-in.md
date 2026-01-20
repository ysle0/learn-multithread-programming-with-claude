# Fan-Out/Fan-In Pattern

## Overview

The Fan-Out/Fan-In pattern distributes work across multiple parallel workers (fan-out) and then aggregates the results (fan-in). This pattern exploits data parallelism to process independent chunks of work concurrently, then combines the results into a final output. It's one of the most effective patterns for achieving scalability.

## Problem Statement

Many computational problems can be divided into independent subtasks:
- Processing large datasets (map-reduce)
- Parallel search across multiple sources
- Distributed computation
- Batch processing of independent items

Processing sequentially wastes available parallelism and takes too long.

## Solution Architecture

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

Characteristics:
- Workers process data in parallel (fan-out)
- No dependencies between workers
- Results combined by aggregator (fan-in)
- Scalability: add more workers
```

## Basic Implementation

### Simple Fan-Out/Fan-In

```cpp
#include <vector>
#include <thread>
#include <future>
#include <numeric>
#include <algorithm>

// Fan-out/Fan-in using futures
template<typename InputIterator, typename Function>
auto parallel_transform(InputIterator begin, InputIterator end,
                       Function func, size_t num_workers)
    -> std::vector<decltype(func(*begin))> {

    using ResultType = decltype(func(*begin));

    size_t total_items = std::distance(begin, end);
    size_t chunk_size = (total_items + num_workers - 1) / num_workers;

    std::vector<std::future<std::vector<ResultType>>> futures;

    // Fan-out: Launch workers
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

    // Fan-in: Collect results
    std::vector<ResultType> combined_results;
    for (auto& future : futures) {
        auto partial_results = future.get();
        combined_results.insert(combined_results.end(),
                               partial_results.begin(),
                               partial_results.end());
    }

    return combined_results;
}

// Example usage
int main() {
    std::vector<int> data(1000);
    std::iota(data.begin(), data.end(), 1);  // 1, 2, 3, ..., 1000

    // Square each number in parallel
    auto results = parallel_transform(data.begin(), data.end(),
        [](int x) { return x * x; },
        std::thread::hardware_concurrency()
    );

    std::cout << "Processed " << results.size() << " items\n";
    return 0;
}
```

## Map-Reduce Pattern

```cpp
template<typename InputIterator, typename MapFunc, typename ReduceFunc>
auto map_reduce(InputIterator begin, InputIterator end,
               MapFunc mapper, ReduceFunc reducer,
               size_t num_workers) {

    using MappedType = decltype(mapper(*begin));

    size_t total_items = std::distance(begin, end);
    size_t chunk_size = (total_items + num_workers - 1) / num_workers;

    std::vector<std::future<std::vector<MappedType>>> futures;

    // Map phase (fan-out)
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

    // Reduce phase (fan-in)
    std::vector<MappedType> all_mapped;
    for (auto& future : futures) {
        auto partial = future.get();
        all_mapped.insert(all_mapped.end(), partial.begin(), partial.end());
    }

    // Final reduction
    if (all_mapped.empty()) {
        throw std::runtime_error("No data to reduce");
    }

    auto result = all_mapped[0];
    for (size_t i = 1; i < all_mapped.size(); ++i) {
        result = reducer(result, all_mapped[i]);
    }

    return result;
}

// Example: Parallel sum
int main() {
    std::vector<int> data(10000);
    std::iota(data.begin(), data.end(), 1);

    auto sum = map_reduce(
        data.begin(), data.end(),
        [](int x) { return x; },           // Map: identity
        [](int a, int b) { return a + b; }, // Reduce: sum
        std::thread::hardware_concurrency()
    );

    std::cout << "Sum: " << sum << "\n";
    return 0;
}
```

## Worker Pool with Fan-Out/Fan-In

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
        // Wait until all tasks are completed
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
                    std::cerr << "Worker error: " << e.what() << "\n";
                }

                active_workers_.fetch_sub(1);
            }
        }
    }
};

// Usage
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

## Advanced: Hierarchical Fan-Out/Fan-In

```cpp
// Tree-based reduction for better scalability
template<typename T, typename ReduceFunc>
T hierarchical_reduce(const std::vector<T>& data, ReduceFunc reducer,
                      size_t num_workers) {
    if (data.empty()) {
        throw std::runtime_error("Cannot reduce empty data");
    }

    if (data.size() == 1) {
        return data[0];
    }

    // Level 1: Fan-out to workers for partial reductions
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

    // Level 2: Collect partial results
    std::vector<T> partial_results;
    for (auto& future : level1_futures) {
        partial_results.push_back(future.get());
    }

    // Level 3: Final reduction (could be recursive for more levels)
    T final_result = partial_results[0];
    for (size_t i = 1; i < partial_results.size(); ++i) {
        final_result = reducer(final_result, partial_results[i]);
    }

    return final_result;
}

// Example: Find maximum in parallel
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

## Scatter-Gather Pattern

```cpp
#include <map>

template<typename Key, typename Value>
class ScatterGather {
public:
    using DataSource = std::function<Value(Key)>;

    ScatterGather(const std::vector<DataSource>& sources)
        : sources_(sources) {}

    // Scatter request to all sources, gather results
    std::map<size_t, Value> query(const Key& key) {
        std::vector<std::future<Value>> futures;

        // Scatter: Query all sources in parallel
        for (size_t i = 0; i < sources_.size(); ++i) {
            futures.push_back(std::async(std::launch::async,
                sources_[i], key
            ));
        }

        // Gather: Collect all results
        std::map<size_t, Value> results;
        for (size_t i = 0; i < futures.size(); ++i) {
            try {
                results[i] = futures[i].get();
            } catch (const std::exception& e) {
                std::cerr << "Source " << i << " failed: "
                          << e.what() << "\n";
            }
        }

        return results;
    }

    // Query with timeout: return partial results
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
                break;  // Timeout
            }

            if (futures[i].wait_for(remaining) == std::future_status::ready) {
                try {
                    results[i] = futures[i].get();
                } catch (const std::exception& e) {
                    std::cerr << "Source " << i << " error: "
                              << e.what() << "\n";
                }
            }
        }

        return results;
    }

private:
    std::vector<DataSource> sources_;
};

// Example: Distributed search
struct SearchResult {
    std::vector<std::string> items;
};

int main() {
    // Simulated data sources
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

    // Query all sources with 250ms timeout
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

## Load Balancing Strategies

### Round-Robin Distribution

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

### Work-Stealing Distribution

```cpp
template<typename Task>
class WorkStealingDistributor {
private:
    std::vector<std::deque<Task>> worker_queues_;
    std::vector<std::mutex> mutexes_;

public:
    explicit WorkStealingDistributor(size_t num_workers)
        : worker_queues_(num_workers), mutexes_(num_workers) {}

    // Worker pushes to own queue
    void push_local(size_t worker_id, Task task) {
        std::lock_guard<std::mutex> lock(mutexes_[worker_id]);
        worker_queues_[worker_id].push_front(std::move(task));
    }

    // Worker steals from other queues when idle
    std::optional<Task> steal(size_t worker_id) {
        // Try stealing from other workers
        for (size_t i = 1; i < worker_queues_.size(); ++i) {
            size_t victim = (worker_id + i) % worker_queues_.size();

            std::lock_guard<std::mutex> lock(mutexes_[victim]);
            if (!worker_queues_[victim].empty()) {
                Task task = std::move(worker_queues_[victim].back());
                worker_queues_[victim].pop_back();
                return task;
            }
        }

        return std::nullopt;  // No work to steal
    }
};
```

## Aggregation Strategies

### Sum Aggregation

```cpp
template<typename T>
T sum_aggregator(const std::vector<T>& results) {
    return std::accumulate(results.begin(), results.end(), T{});
}
```

### Min/Max Aggregation

```cpp
template<typename T>
T max_aggregator(const std::vector<T>& results) {
    return *std::max_element(results.begin(), results.end());
}
```

### Custom Aggregation

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

## Real-World Applications

### 1. Parallel Image Processing

```cpp
// Fan-out: Process image tiles in parallel
// Fan-in: Stitch tiles back together
std::vector<ImageTile> tiles = split_image(large_image);

auto processed_tiles = parallel_transform(tiles.begin(), tiles.end(),
    [](const ImageTile& tile) {
        return apply_filter(tile);
    },
    num_workers
);

Image result = stitch_tiles(processed_tiles);
```

### 2. Distributed Search

```
Query ───┬──▶ Database 1 ──┐
         ├──▶ Database 2 ──┤
         ├──▶ Database 3 ──┼──▶ Merge Results
         └──▶ Database N ──┘
```

### 3. Web Crawling

```cpp
// Fan-out: Fetch multiple URLs in parallel
// Fan-in: Aggregate extracted data
std::vector<std::string> urls = get_urls_to_crawl();

auto pages = parallel_transform(urls.begin(), urls.end(),
    [](const std::string& url) {
        return fetch_and_parse(url);
    },
    100  // 100 concurrent fetches
);

auto all_links = merge_links(pages);
```

### 4. Batch Data Processing

```cpp
// Process large dataset in parallel chunks
std::vector<DataChunk> chunks = partition_data(huge_dataset, num_workers);

auto results = parallel_transform(chunks.begin(), chunks.end(),
    [](const DataChunk& chunk) {
        return process_chunk(chunk);
    },
    num_workers
);

auto final_result = reduce_results(results);
```

### 5. Financial Portfolio Analysis

```cpp
// Analyze multiple stocks in parallel
std::vector<Stock> portfolio = get_portfolio();

auto analyses = parallel_transform(portfolio.begin(), portfolio.end(),
    [](const Stock& stock) {
        return calculate_risk_metrics(stock);
    },
    num_workers
);

PortfolioRisk total_risk = aggregate_risk(analyses);
```

## Performance Considerations

### Amdahl's Law

```
Speedup = 1 / ((1 - P) + P/N)

Where:
  P = Parallel portion (0 to 1)
  N = Number of workers

Example:
  90% parallelizable (P=0.9), 10 workers:
  Speedup = 1 / (0.1 + 0.9/10) = 5.26x

  Perfect parallelization (P=1.0), 10 workers:
  Speedup = 10x
```

### Overhead Analysis

```cpp
// Measure overhead
auto start = std::chrono::high_resolution_clock::now();

// Sequential
int sum_seq = std::accumulate(data.begin(), data.end(), 0);

auto seq_time = std::chrono::high_resolution_clock::now() - start;

// Parallel
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

### Optimal Worker Count

```cpp
// For CPU-bound tasks
size_t optimal_workers = std::thread::hardware_concurrency();

// For I/O-bound tasks (can be much higher)
size_t optimal_workers = std::thread::hardware_concurrency() * 10;

// For mixed workloads (benchmark to find optimal)
for (size_t workers = 1; workers <= 32; workers *= 2) {
    auto time = benchmark_with_workers(workers);
    std::cout << "Workers: " << workers << ", Time: " << time << "\n";
}
```

## Common Pitfalls

### 1. Overhead Dominates for Small Tasks

```cpp
// BAD: Parallel overhead > work
std::vector<int> small_data = {1, 2, 3};
auto result = parallel_sum(small_data);  // Slower than sequential!

// GOOD: Only parallelize large enough workloads
if (data.size() > 1000) {
    result = parallel_sum(data);
} else {
    result = sequential_sum(data);
}
```

### 2. Load Imbalance

```cpp
// BAD: Uneven work distribution
// Worker 1: 1000 items
// Worker 2: 10 items
// Worker 2 finishes early, sits idle

// GOOD: Balance workload or use work stealing
```

### 3. Memory Contention

```cpp
// BAD: All workers writing to same location
std::atomic<int> counter{0};
parallel_for_each([&](int x) {
    counter++;  // Contention!
});

// GOOD: Per-worker accumulation, then aggregate
std::vector<int> per_worker_counts(num_workers, 0);
// Each worker updates its own counter
// Final fan-in sums all counters
```

## Testing Strategies

### Correctness

```cpp
// Verify parallel result matches sequential
auto seq_result = sequential_process(data);
auto par_result = parallel_process(data);
assert(seq_result == par_result);
```

### Scalability

```cpp
// Measure speedup with increasing workers
for (size_t workers = 1; workers <= 16; workers *= 2) {
    auto time = benchmark(data, workers);
    double speedup = baseline_time / time;
    std::cout << "Workers: " << workers
              << ", Speedup: " << speedup << "x\n";
}
```

## Pros and Cons

### Pros
- Excellent scalability for data-parallel problems
- Simple mental model
- Easy to implement with futures/async
- Near-linear speedup for embarrassingly parallel problems
- Good CPU utilization

### Cons
- Overhead for small tasks
- Potential load imbalance
- Memory overhead for duplicating data
- Not suitable for dependencies between tasks
- Aggregation can become bottleneck

## Best Practices

1. **Ensure tasks are independent** (no shared state)
2. **Size tasks appropriately** (balance overhead vs parallelism)
3. **Balance workload** across workers
4. **Use appropriate number of workers** (profile to find optimal)
5. **Handle errors in workers** (don't let one failure kill all)
6. **Consider hierarchical fan-in** for large result sets
7. **Monitor worker utilization** to detect load imbalance

## Summary

Fan-Out/Fan-In is ideal for:
- Data-parallel problems
- Embarrassingly parallel workloads
- Batch processing
- Distributed computation

It's one of the most straightforward and effective patterns for achieving parallelism, providing near-linear speedup when applied to suitable problems.
