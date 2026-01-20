# Thread Pool Pattern

## Overview

The Thread Pool pattern manages a pool of reusable worker threads that execute tasks from a queue. Instead of creating a new thread for each task (expensive and unscalable), threads are created once and reused for multiple tasks. This pattern is fundamental for building scalable concurrent applications.

## Problem Statement

Creating threads on-demand has significant drawbacks:
- Thread creation/destruction overhead (system calls, memory allocation)
- Resource exhaustion with too many threads
- Context switching overhead degrades performance
- Difficult to control level of concurrency
- Poor cache locality from thread churn

## Solution Architecture

```
                    ┌─────────────────────────┐
                    │     Task Queue          │
    submit(task) ──▶│  [T1][T2][T3][T4][T5]  │
                    └─────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
              ┌─────▼─────┐       ┌────▼──────┐
              │ Worker 1  │  ...  │ Worker N  │
              │ (Thread)  │       │ (Thread)  │
              └───────────┘       └───────────┘
                    │                   │
              ┌─────▼─────┐       ┌────▼──────┐
              │Execute T1 │       │Execute T2 │
              └───────────┘       └───────────┘

Lifecycle:
1. Initialize pool with N worker threads
2. Workers wait on task queue
3. Submit tasks to queue
4. Workers execute tasks
5. Workers return to waiting state
6. Shutdown: complete tasks and join threads
```

## Basic Implementation

### Simple Thread Pool

```cpp
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <future>
#include <memory>

class ThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;

    std::mutex mutex_;
    std::condition_variable condition_;
    bool stop_ = false;

public:
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                worker_thread();
            });
        }
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        condition_.notify_all();

        for (auto& worker : workers_) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }

    // Submit task and get future
    template<typename F, typename... Args>
    auto submit(F&& f, Args&&... args)
        -> std::future<typename std::invoke_result_t<F, Args...>> {

        using return_type = typename std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<return_type()>>(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

        std::future<return_type> result = task->get_future();

        {
            std::unique_lock<std::mutex> lock(mutex_);

            if (stop_) {
                throw std::runtime_error("Cannot submit to stopped pool");
            }

            tasks_.emplace([task]() { (*task)(); });
        }

        condition_.notify_one();
        return result;
    }

    size_t size() const {
        return workers_.size();
    }

private:
    void worker_thread() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(mutex_);

                condition_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });

                if (stop_ && tasks_.empty()) {
                    return;
                }

                task = std::move(tasks_.front());
                tasks_.pop();
            }

            task();
        }
    }
};
```

### Usage Example

```cpp
#include <iostream>
#include <chrono>

int compute_factorial(int n) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

int main() {
    ThreadPool pool(4);  // 4 worker threads

    std::vector<std::future<int>> results;

    // Submit 10 tasks
    for (int i = 1; i <= 10; ++i) {
        results.push_back(pool.submit(compute_factorial, i));
    }

    // Collect results
    for (size_t i = 0; i < results.size(); ++i) {
        std::cout << "Factorial(" << (i + 1) << ") = "
                  << results[i].get() << "\n";
    }

    return 0;
}  // Pool destructor waits for all tasks to complete
```

## Advanced Features

### Thread Pool with Priority Queues

```cpp
#include <set>

class PriorityThreadPool {
private:
    struct Task {
        int priority;
        std::function<void()> func;
        size_t sequence;  // For FIFO within same priority

        bool operator<(const Task& other) const {
            if (priority != other.priority) {
                return priority > other.priority;  // Higher priority first
            }
            return sequence < other.sequence;  // FIFO for same priority
        }
    };

    std::vector<std::thread> workers_;
    std::set<Task> tasks_;
    std::mutex mutex_;
    std::condition_variable condition_;
    bool stop_ = false;
    size_t next_sequence_ = 0;

public:
    explicit PriorityThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] { worker_thread(); });
        }
    }

    ~PriorityThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        condition_.notify_all();

        for (auto& worker : workers_) {
            worker.join();
        }
    }

    template<typename F>
    void submit(F&& f, int priority = 0) {
        {
            std::unique_lock<std::mutex> lock(mutex_);

            if (stop_) {
                throw std::runtime_error("Pool is stopped");
            }

            tasks_.insert(Task{
                priority,
                std::forward<F>(f),
                next_sequence_++
            });
        }
        condition_.notify_one();
    }

private:
    void worker_thread() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                condition_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });

                if (stop_ && tasks_.empty()) {
                    return;
                }

                auto it = tasks_.begin();
                task = std::move(it->func);
                tasks_.erase(it);
            }

            task();
        }
    }
};
```

### Thread Pool with Work Stealing

```cpp
#include <deque>
#include <random>

class WorkStealingThreadPool {
private:
    struct WorkerData {
        std::deque<std::function<void()>> queue;
        std::mutex mutex;
    };

    std::vector<std::thread> workers_;
    std::vector<std::unique_ptr<WorkerData>> worker_data_;
    std::atomic<bool> stop_{false};
    std::atomic<size_t> next_worker_{0};

public:
    explicit WorkStealingThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            worker_data_.push_back(std::make_unique<WorkerData>());
        }

        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this, i] {
                worker_thread(i);
            });
        }
    }

    ~WorkStealingThreadPool() {
        stop_.store(true);

        for (auto& worker : workers_) {
            worker.join();
        }
    }

    template<typename F>
    void submit(F&& f) {
        size_t target = next_worker_.fetch_add(1) % workers_.size();

        std::lock_guard<std::mutex> lock(worker_data_[target]->mutex);
        worker_data_[target]->queue.push_back(std::forward<F>(f));
    }

private:
    void worker_thread(size_t worker_id) {
        std::random_device rd;
        std::mt19937 gen(rd());

        while (!stop_.load()) {
            std::function<void()> task;

            // Try to get task from own queue
            {
                std::lock_guard<std::mutex> lock(worker_data_[worker_id]->mutex);
                if (!worker_data_[worker_id]->queue.empty()) {
                    task = std::move(worker_data_[worker_id]->queue.front());
                    worker_data_[worker_id]->queue.pop_front();
                }
            }

            // If own queue is empty, try stealing from others
            if (!task) {
                std::uniform_int_distribution<> dis(0, workers_.size() - 1);

                for (size_t i = 0; i < workers_.size(); ++i) {
                    size_t victim = dis(gen);
                    if (victim == worker_id) continue;

                    std::lock_guard<std::mutex> lock(worker_data_[victim]->mutex);
                    if (!worker_data_[victim]->queue.empty()) {
                        // Steal from back (oldest task)
                        task = std::move(worker_data_[victim]->queue.back());
                        worker_data_[victim]->queue.pop_back();
                        break;
                    }
                }
            }

            if (task) {
                task();
            } else {
                // No work found, sleep briefly
                std::this_thread::sleep_for(std::chrono::microseconds(100));
            }
        }
    }
};
```

### Thread Pool with Dynamic Sizing

```cpp
class DynamicThreadPool {
private:
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;

    std::mutex mutex_;
    std::condition_variable condition_;
    bool stop_ = false;

    size_t min_threads_;
    size_t max_threads_;
    std::chrono::seconds idle_timeout_;

    std::atomic<size_t> active_workers_{0};
    std::atomic<size_t> total_workers_{0};

public:
    DynamicThreadPool(size_t min_threads, size_t max_threads,
                      std::chrono::seconds idle_timeout = std::chrono::seconds(60))
        : min_threads_(min_threads),
          max_threads_(max_threads),
          idle_timeout_(idle_timeout) {

        for (size_t i = 0; i < min_threads_; ++i) {
            add_worker();
        }
    }

    ~DynamicThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        condition_.notify_all();

        for (auto& worker : workers_) {
            if (worker.joinable()) {
                worker.join();
            }
        }
    }

    template<typename F>
    void submit(F&& f) {
        {
            std::unique_lock<std::mutex> lock(mutex_);

            if (stop_) {
                throw std::runtime_error("Pool is stopped");
            }

            tasks_.push(std::forward<F>(f));

            // Spawn new worker if all busy and under max
            if (active_workers_.load() >= total_workers_.load() &&
                total_workers_.load() < max_threads_) {
                add_worker();
            }
        }

        condition_.notify_one();
    }

    size_t active_count() const { return active_workers_.load(); }
    size_t total_count() const { return total_workers_.load(); }

private:
    void add_worker() {
        workers_.emplace_back([this] { worker_thread(); });
        total_workers_.fetch_add(1);
    }

    void worker_thread() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(mutex_);

                bool timed_out = !condition_.wait_for(lock, idle_timeout_,
                    [this] { return stop_ || !tasks_.empty(); });

                // Allow thread to exit if idle too long and above minimum
                if (timed_out && total_workers_.load() > min_threads_) {
                    total_workers_.fetch_sub(1);
                    return;
                }

                if (stop_ && tasks_.empty()) {
                    total_workers_.fetch_sub(1);
                    return;
                }

                if (tasks_.empty()) {
                    continue;
                }

                task = std::move(tasks_.front());
                tasks_.pop();
            }

            active_workers_.fetch_add(1);
            task();
            active_workers_.fetch_sub(1);
        }
    }
};
```

## Complete Example: Parallel File Processor

```cpp
#include <iostream>
#include <fstream>
#include <filesystem>
#include <vector>
#include <string>

namespace fs = std::filesystem;

// Process a single file
std::string process_file(const fs::path& path) {
    std::ifstream file(path);
    std::string content((std::istreambuf_iterator<char>(file)),
                        std::istreambuf_iterator<char>());

    // Simulate processing
    std::this_thread::sleep_for(std::chrono::milliseconds(100));

    return path.filename().string() + ": " +
           std::to_string(content.size()) + " bytes";
}

int main() {
    ThreadPool pool(std::thread::hardware_concurrency());

    std::vector<std::future<std::string>> results;

    // Find all .txt files
    for (const auto& entry : fs::directory_iterator(".")) {
        if (entry.path().extension() == ".txt") {
            results.push_back(pool.submit(process_file, entry.path()));
        }
    }

    // Collect and print results
    for (auto& result : results) {
        std::cout << result.get() << "\n";
    }

    return 0;
}
```

## Performance Considerations

### Optimal Thread Count

```cpp
// CPU-bound tasks: number of cores
size_t cpu_bound_size = std::thread::hardware_concurrency();

// I/O-bound tasks: more threads to hide latency
size_t io_bound_size = std::thread::hardware_concurrency() * 2;

// Mixed workload: tune based on profiling
size_t mixed_size = std::thread::hardware_concurrency() * 1.5;
```

### Task Granularity

```
Too Fine Grained:
  - High scheduling overhead
  - Poor cache utilization
  - Mutex contention

Optimal:
  - Balance overhead vs parallelism
  - ~1ms - 100ms per task

Too Coarse Grained:
  - Load imbalance
  - Idle workers
  - Poor utilization
```

### Benchmarking

```cpp
template<typename PoolType>
void benchmark_pool(const std::string& name, size_t num_tasks) {
    PoolType pool(std::thread::hardware_concurrency());

    auto start = std::chrono::high_resolution_clock::now();

    std::vector<std::future<void>> futures;
    for (size_t i = 0; i < num_tasks; ++i) {
        futures.push_back(pool.submit([i] {
            // Simulate work
            std::this_thread::sleep_for(std::chrono::microseconds(100));
        }));
    }

    for (auto& f : futures) {
        f.get();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    std::cout << name << ": " << ms << "ms for "
              << num_tasks << " tasks\n";
}
```

## Common Pitfalls

### 1. Blocking the Pool

```cpp
// BAD: Deadlock if pool size < 2
ThreadPool pool(1);
auto future1 = pool.submit([] {
    // This task waits for another task...
    auto future2 = pool.submit([] { return 42; });
    return future2.get();  // DEADLOCK!
});
```

### 2. Exception Handling

```cpp
// BAD: Uncaught exceptions terminate the program
pool.submit([] {
    throw std::runtime_error("Error!");  // Terminates!
});

// GOOD: Catch exceptions
pool.submit([] {
    try {
        // ... work that might throw
    } catch (const std::exception& e) {
        // Handle error
    }
});

// BETTER: Use futures to propagate exceptions
auto future = pool.submit([] {
    throw std::runtime_error("Error!");
});

try {
    future.get();  // Re-throws exception
} catch (const std::exception& e) {
    // Handle error
}
```

### 3. Shared State Races

```cpp
// BAD: Race condition
int counter = 0;
for (int i = 0; i < 100; ++i) {
    pool.submit([&counter] {
        counter++;  // RACE!
    });
}

// GOOD: Use atomic or return results
std::atomic<int> counter{0};
for (int i = 0; i < 100; ++i) {
    pool.submit([&counter] {
        counter.fetch_add(1);
    });
}
```

### 4. Resource Leaks

```cpp
// BAD: Pool destroyed before tasks complete
{
    ThreadPool pool(4);
    pool.submit(long_running_task);
}  // Pool destroyed immediately!

// GOOD: Wait for tasks via futures
{
    ThreadPool pool(4);
    auto future = pool.submit(long_running_task);
    future.get();  // Wait for completion
}
```

## Real-World Applications

### 1. Web Servers
```
HTTP requests → Thread Pool → Request handlers
```

### 2. Database Connection Pools
```
Queries → Thread Pool → DB connections
```

### 3. Parallel Compilation
```
Source files → Thread Pool → Compile tasks
```

### 4. Image Processing
```
Images → Thread Pool → Resize/filter tasks
```

### 5. Background Job Processing
```
Jobs → Thread Pool → Job executors
```

## Advanced Patterns

### Task Dependencies

```cpp
class TaskGraph {
    // Execute tasks respecting dependencies
    // A → B (B depends on A)
    // A → C (C depends on A)
};
```

### Nested Parallelism

```cpp
// Outer parallel loop
for (auto& chunk : chunks) {
    pool.submit([&chunk] {
        // Inner parallel operations
        process_chunk(chunk);
    });
}
```

### Cancellation

```cpp
class CancellableTask {
    std::atomic<bool> cancelled_{false};

    void execute() {
        while (!cancelled_) {
            // Do work...
            if (should_cancel()) {
                cancelled_ = true;
            }
        }
    }
};
```

## Pros and Cons

### Pros
- Reuses threads, avoiding creation/destruction overhead
- Limits concurrent execution, preventing resource exhaustion
- Decouples task submission from execution
- Easy to use and understand
- Good CPU utilization for CPU-bound tasks

### Cons
- Fixed overhead per task (queuing, synchronization)
- Not suitable for very fine-grained tasks
- Can cause deadlocks if tasks wait on each other
- Memory overhead for queue and threads
- Doesn't help with I/O-bound tasks (consider async I/O instead)

## Best Practices

1. **Size pool based on workload**:
   - CPU-bound: `hardware_concurrency()`
   - I/O-bound: `2 * hardware_concurrency()` or more
   - Mixed: Profile and tune

2. **Return futures for results**:
   ```cpp
   auto result = pool.submit(compute);
   process(result.get());
   ```

3. **Use RAII for pool lifecycle**:
   ```cpp
   {
       ThreadPool pool(4);
       // Submit tasks...
   }  // Automatic cleanup
   ```

4. **Avoid blocking in tasks**:
   - Don't wait on other pool tasks
   - Don't hold locks for long periods
   - Don't do blocking I/O (use async I/O)

5. **Monitor pool health**:
   - Queue depth
   - Worker utilization
   - Task completion time

6. **Consider specialized pools**:
   - Separate pools for different task types
   - Priority pools for critical tasks
   - Work-stealing for load balancing

## Testing Strategies

### Stress Test
```cpp
// Submit many tasks quickly
for (int i = 0; i < 10000; ++i) {
    pool.submit(quick_task);
}
```

### Correctness Test
```cpp
// Verify all tasks execute exactly once
std::atomic<int> counter{0};
for (int i = 0; i < 1000; ++i) {
    pool.submit([&] { counter++; });
}
// Wait and verify counter == 1000
```

### Exception Safety Test
```cpp
// Verify exceptions don't crash workers
pool.submit([] { throw std::runtime_error("test"); });
// Pool should still function
```

## Summary

Thread pools are essential for:
- Scalable concurrent task execution
- Resource management and limiting
- Decoupling task creation from execution
- Building responsive applications

Use thread pools when you have many independent tasks and want to control concurrency levels efficiently.
