# Pipeline Pattern

## Overview

The Pipeline pattern processes data through a series of sequential stages, where each stage performs a specific transformation. Stages run concurrently, with each stage processing different items simultaneously. This pattern is ideal for stream processing where data flows through multiple processing steps.

## Problem Statement

Many data processing tasks involve multiple sequential transformations:
- Video encoding (decode → transform → encode)
- Image processing (load → filter → resize → save)
- Data ETL (extract → transform → load)
- Compilation (parse → optimize → codegen)

Sequential processing wastes CPU time. The pipeline pattern enables parallelism across stages.

## Solution Architecture

```
Input Stream
     │
     ▼
┌────────────┐         ┌────────────┐         ┌────────────┐
│  Stage 1   │──queue─▶│  Stage 2   │──queue─▶│  Stage 3   │
│  (Thread)  │         │  (Thread)  │         │  (Thread)  │
└────────────┘         └────────────┘         └────────────┘
     │                      │                      │
   Item A                 Item B                 Item C
                                                    │
                                                    ▼
                                              Output Stream

Parallelism:
- Stage 1 processes Item D
- Stage 2 processes Item B (already processed by Stage 1)
- Stage 3 processes Item C (already processed by Stages 1 and 2)

Throughput = min(throughput of each stage)
Latency = sum(latency of each stage)
```

## Basic Implementation

### Simple Pipeline

```cpp
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <optional>
#include <vector>

template<typename Input, typename Output>
class Stage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    Stage<Output, void>* next_stage_ = nullptr;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

public:
    explicit Stage(std::function<Output(Input)> processor)
        : processor_(std::move(processor)) {}

    void set_next(Stage<Output, void>* next) {
        next_stage_ = next;
    }

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    // Notify next stage to stop
                    if (next_stage_) {
                        next_stage_->stop();
                    }
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            // Process item
            try {
                Output result = processor_(std::move(item));

                // Pass to next stage
                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "Stage error: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }

    size_t queue_size() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return input_queue_.size();
    }
};

// Specialization for terminal stage (no output)
template<typename Input>
class Stage<Input, void> {
private:
    std::function<void(Input)> processor_;
    std::queue<Input> input_queue_;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

public:
    explicit Stage(std::function<void(Input)> processor)
        : processor_(std::move(processor)) {}

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            try {
                processor_(std::move(item));
            } catch (const std::exception& e) {
                std::cerr << "Terminal stage error: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }
};
```

### Complete Example: Image Processing Pipeline

```cpp
#include <iostream>
#include <vector>
#include <string>
#include <chrono>

// Image data structure
struct Image {
    int id;
    std::string filename;
    std::vector<uint8_t> data;
    int width, height;

    Image(int id, const std::string& name, int w, int h)
        : id(id), filename(name), width(w), height(h),
          data(w * h * 3) {}  // RGB
};

// Stage 1: Load image from disk
Image load_image(const std::string& filename) {
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "Loaded: " << filename << "\n";
    return Image(0, filename, 1920, 1080);
}

// Stage 2: Apply filter
Image apply_filter(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "Filtered: " << img.filename << "\n";
    // Apply filter to img.data
    return img;
}

// Stage 3: Resize
Image resize_image(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(75));
    std::cout << "Resized: " << img.filename << "\n";
    img.width /= 2;
    img.height /= 2;
    img.data.resize(img.width * img.height * 3);
    return img;
}

// Stage 4: Save to disk
void save_image(Image img) {
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::cout << "Saved: " << img.filename << "\n";
}

int main() {
    // Create pipeline stages
    Stage<std::string, Image> loader(load_image);
    Stage<Image, Image> filter(apply_filter);
    Stage<Image, Image> resizer(resize_image);
    Stage<Image, void> saver(save_image);

    // Connect stages
    loader.set_next(&filter);
    filter.set_next(&resizer);
    resizer.set_next(&saver);

    // Start stage threads
    std::thread t1([&] { loader.run(); });
    std::thread t2([&] { filter.run(); });
    std::thread t3([&] { resizer.run(); });
    std::thread t4([&] { saver.run(); });

    // Feed input
    std::vector<std::string> files = {
        "image1.jpg", "image2.jpg", "image3.jpg",
        "image4.jpg", "image5.jpg"
    };

    for (const auto& file : files) {
        loader.push(file);
    }

    // Allow processing
    std::this_thread::sleep_for(std::chrono::seconds(3));

    // Shutdown pipeline
    loader.stop();

    t1.join();
    t2.join();
    t3.join();
    t4.join();

    std::cout << "Pipeline completed\n";
    return 0;
}
```

## Advanced Implementation: Generic Pipeline Builder

```cpp
template<typename T>
class Pipeline {
private:
    struct StageBase {
        virtual ~StageBase() = default;
        virtual void start() = 0;
        virtual void stop() = 0;
        virtual void join() = 0;
    };

    template<typename In, typename Out>
    struct StageImpl : StageBase {
        Stage<In, Out> stage;
        std::thread thread;

        StageImpl(std::function<Out(In)> func) : stage(func) {}

        void start() override {
            thread = std::thread([this] { stage.run(); });
        }

        void stop() override {
            stage.stop();
        }

        void join() override {
            if (thread.joinable()) {
                thread.join();
            }
        }
    };

    std::vector<std::unique_ptr<StageBase>> stages_;
    StageBase* first_stage_ = nullptr;

public:
    template<typename In, typename Out>
    Pipeline& add_stage(std::function<Out(In)> processor) {
        auto stage_impl = std::make_unique<StageImpl<In, Out>>(processor);

        if (!first_stage_) {
            first_stage_ = stage_impl.get();
        }

        // Connect to previous stage
        if (!stages_.empty()) {
            // Type-safe connection would require more template magic
            // This is a simplified version
        }

        stages_.push_back(std::move(stage_impl));
        return *this;
    }

    void start() {
        for (auto& stage : stages_) {
            stage->start();
        }
    }

    void stop() {
        if (!stages_.empty()) {
            stages_[0]->stop();
        }
    }

    void join() {
        for (auto& stage : stages_) {
            stage->join();
        }
    }

    template<typename In>
    void push(In item) {
        // Push to first stage
        // Requires casting, simplified here
    }
};
```

## Parallel Stages (Fan-Out within Stage)

```cpp
template<typename Input, typename Output>
class ParallelStage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    size_t num_workers_;

    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;

    Stage<Output, void>* next_stage_ = nullptr;

public:
    ParallelStage(std::function<Output(Input)> processor, size_t num_workers)
        : processor_(std::move(processor)), num_workers_(num_workers) {}

    void set_next(Stage<Output, void>* next) {
        next_stage_ = next;
    }

    void push(Input item) {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            input_queue_.push(std::move(item));
        }
        cv_.notify_one();
    }

    void run_worker() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            try {
                Output result = processor_(std::move(item));

                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "Worker error: " << e.what() << "\n";
            }
        }
    }

    std::vector<std::thread> start() {
        std::vector<std::thread> workers;
        for (size_t i = 0; i < num_workers_; ++i) {
            workers.emplace_back([this] { run_worker(); });
        }
        return workers;
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }
};
```

## Backpressure Handling

```cpp
template<typename Input, typename Output>
class BoundedStage {
private:
    std::function<Output(Input)> processor_;
    std::queue<Input> input_queue_;
    size_t max_queue_size_;

    mutable std::mutex mutex_;
    std::condition_variable input_cv_;
    std::condition_variable capacity_cv_;
    bool stopped_ = false;

    Stage<Output, void>* next_stage_ = nullptr;

public:
    BoundedStage(std::function<Output(Input)> processor, size_t max_queue_size)
        : processor_(std::move(processor)),
          max_queue_size_(max_queue_size) {}

    // Blocking push with backpressure
    bool push(Input item) {
        std::unique_lock<std::mutex> lock(mutex_);

        // Wait for capacity
        capacity_cv_.wait(lock, [this] {
            return input_queue_.size() < max_queue_size_ || stopped_;
        });

        if (stopped_) {
            return false;
        }

        input_queue_.push(std::move(item));
        input_cv_.notify_one();
        return true;
    }

    void run() {
        while (true) {
            Input item;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                input_cv_.wait(lock, [this] {
                    return !input_queue_.empty() || stopped_;
                });

                if (stopped_ && input_queue_.empty()) {
                    if (next_stage_) {
                        next_stage_->stop();
                    }
                    break;
                }

                if (input_queue_.empty()) {
                    continue;
                }

                item = std::move(input_queue_.front());
                input_queue_.pop();
            }

            // Notify that capacity is available
            capacity_cv_.notify_one();

            try {
                Output result = processor_(std::move(item));

                if (next_stage_) {
                    next_stage_->push(std::move(result));
                }
            } catch (const std::exception& e) {
                std::cerr << "Stage error: " << e.what() << "\n";
            }
        }
    }

    void stop() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stopped_ = true;
        }
        input_cv_.notify_all();
        capacity_cv_.notify_all();
    }
};
```

## Performance Monitoring

```cpp
template<typename Input, typename Output>
class MonitoredStage : public Stage<Input, Output> {
private:
    std::atomic<size_t> items_processed_{0};
    std::atomic<size_t> total_processing_time_ms_{0};
    std::chrono::steady_clock::time_point start_time_;

public:
    MonitoredStage(std::function<Output(Input)> processor)
        : Stage<Input, Output>(
            [this, processor](Input item) {
                auto start = std::chrono::steady_clock::now();

                Output result = processor(std::move(item));

                auto end = std::chrono::steady_clock::now();
                auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
                    end - start).count();

                items_processed_.fetch_add(1);
                total_processing_time_ms_.fetch_add(ms);

                return result;
            }
        ),
        start_time_(std::chrono::steady_clock::now()) {}

    void print_stats() const {
        auto now = std::chrono::steady_clock::now();
        auto elapsed_sec = std::chrono::duration_cast<std::chrono::seconds>(
            now - start_time_).count();

        size_t processed = items_processed_.load();
        size_t total_time = total_processing_time_ms_.load();

        std::cout << "Stage Statistics:\n"
                  << "  Items processed: " << processed << "\n"
                  << "  Throughput: " << (processed / std::max(1L, elapsed_sec))
                  << " items/sec\n"
                  << "  Avg processing time: "
                  << (processed > 0 ? total_time / processed : 0) << "ms\n"
                  << "  Queue size: " << this->queue_size() << "\n";
    }
};
```

## Real-World Applications

### 1. Video Processing

```
┌──────┐    ┌────────┐    ┌──────────┐    ┌────────┐    ┌──────┐
│Decode│───▶│Denoise │───▶│Color Adj.│───▶│ Encode │───▶│ Save │
└──────┘    └────────┘    └──────────┘    └────────┘    └──────┘
  (GPU)      (CPU 4x)         (GPU)         (CPU 8x)     (Disk)
```

### 2. ETL (Extract, Transform, Load)

```
┌────────┐    ┌─────────┐    ┌──────────┐    ┌──────┐
│Extract │───▶│Transform│───▶│ Validate │───▶│ Load │
│  (DB)  │    │ (CPU)   │    │  (CPU)   │    │ (DB) │
└────────┘    └─────────┘    └──────────┘    └──────┘
```

### 3. Log Processing

```
┌──────┐    ┌──────┐    ┌──────────┐    ┌───────────┐
│ Read │───▶│Parse │───▶│ Analyze  │───▶│Aggregate  │
│(Disk)│    │(CPU) │    │  (CPU)   │    │(Memory/DB)│
└──────┘    └──────┘    └──────────┘    └───────────┘
```

### 4. Web Scraping

```
┌──────┐    ┌───────┐    ┌─────────┐    ┌───────┐
│Fetch │───▶│ Parse │───▶│ Extract │───▶│ Store │
│(HTTP)│    │(HTML) │    │  (Data) │    │  (DB) │
└──────┘    └───────┘    └─────────┘    └───────┘
```

## Pipeline Variants

### 1. Branching Pipeline

```
         ┌──────────┐
    ┌───▶│  Stage B │
    │    └──────────┘
┌───┴──┐
│Stage A│
└───┬──┘
    │    ┌──────────┐
    └───▶│  Stage C │
         └──────────┘
```

### 2. Merging Pipeline

```
┌──────────┐
│  Stage A │───┐
└──────────┘   │
               ├───▶┌──────────┐
┌──────────┐   │    │  Stage C │
│  Stage B │───┘    └──────────┘
└──────────┘
```

### 3. Cyclic Pipeline (Feedback)

```
┌──────────┐    ┌──────────┐
│  Stage A │───▶│  Stage B │
└──────────┘    └──────────┘
      ▲               │
      └───────────────┘
       (retry/refine)
```

## Common Pitfalls

### 1. Unbalanced Stages

```cpp
// BAD: Slow stage becomes bottleneck
Stage 1: 10ms  ───┐
Stage 2: 100ms ───┼── Throughput limited by Stage 2
Stage 3: 10ms  ───┘

// GOOD: Parallelize slow stage
Stage 1: 10ms     ───┐
Stage 2: 100ms (4x) ─┼── Balanced throughput
Stage 3: 10ms     ───┘
```

### 2. Unbounded Queues

```cpp
// BAD: Fast producer, slow consumer
// Queue grows without bound → OOM

// GOOD: Bounded queues with backpressure
BoundedStage stage(processor, 100);  // Max 100 items
```

### 3. No Error Handling

```cpp
// BAD: Exception kills stage thread
Output result = processor(item);  // Might throw!

// GOOD: Catch and handle errors
try {
    Output result = processor(item);
} catch (const std::exception& e) {
    // Log error, skip item, or retry
}
```

## Performance Considerations

### Latency vs Throughput

```
Latency: Time for one item to go through entire pipeline
  = Stage1_time + Stage2_time + Stage3_time

Throughput: Items processed per second
  = 1 / max(Stage1_time, Stage2_time, Stage3_time)

Example:
  Stage 1: 100ms
  Stage 2: 200ms  (bottleneck)
  Stage 3: 100ms

  Latency: 400ms per item
  Throughput: 5 items/sec (limited by Stage 2)
```

### Buffer Sizing

```cpp
// Too small: Frequent blocking
const size_t BUFFER_SIZE = 1;  // Stages wait often

// Too large: Memory waste, high latency
const size_t BUFFER_SIZE = 10000;  // Lots of items queued

// Optimal: Balance memory and blocking
const size_t BUFFER_SIZE = 10-100;  // Sweet spot for most cases
```

## Testing Strategies

### Correctness Test

```cpp
// Verify all items processed exactly once
std::atomic<int> counter{0};
auto terminal = Stage<int, void>([&](int x) {
    counter.fetch_add(1);
});

// Feed N items, verify counter == N
```

### Stress Test

```cpp
// High-volume test
for (int i = 0; i < 1000000; ++i) {
    pipeline.push(i);
}
```

### Bottleneck Identification

```cpp
// Monitor queue sizes
std::cout << "Stage 1 queue: " << stage1.queue_size() << "\n";
std::cout << "Stage 2 queue: " << stage2.queue_size() << "\n";
// Growing queue indicates bottleneck downstream
```

## Pros and Cons

### Pros
- Natural parallelism across stages
- Good CPU utilization
- Scalable (add more stages or parallelize stages)
- Clear separation of concerns
- Easy to reason about data flow

### Cons
- Latency increases with number of stages
- Throughput limited by slowest stage
- Queue overhead and memory usage
- Complex error handling across stages
- Debugging can be challenging

## Best Practices

1. **Balance stage processing times**
2. **Use bounded queues** to prevent memory issues
3. **Monitor performance** of each stage
4. **Handle errors gracefully** (don't kill pipeline)
5. **Parallelize slow stages** to balance throughput
6. **Keep stages independent** (no shared state)
7. **Implement backpressure** for slow consumers

## Summary

Pipeline pattern is ideal for:
- Stream processing
- Sequential data transformations
- ETL workloads
- Media processing

It provides excellent throughput by keeping all stages busy simultaneously, while maintaining clear separation between processing steps.
