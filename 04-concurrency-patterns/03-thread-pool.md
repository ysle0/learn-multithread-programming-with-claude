# Thread Pool (스레드 풀) 패턴

## 📌 개요

**Thread Pool**은 작업을 처리하기 위해 미리 생성된 스레드들을 재사용하는 패턴입니다. 매번 스레드를 생성/삭제하는 대신, 고정된 수의 워커 스레드가 작업 큐에서 태스크를 가져와 실행합니다.

**핵심 아이디어**: 스레드 생성 비용을 줄이고, 동시 실행 스레드 수를 제어하여 시스템 리소스를 효율적으로 관리합니다.

---

## 🔍 문제 정의

### 매번 스레드 생성 시 문제점

```cpp
// ❌ 나쁜 예: 요청마다 스레드 생성
void handle_request(Request req) {
    std::thread t([req]() {
        process(req);
    });
    t.detach();  // 수천 개 스레드 생성 → 시스템 과부하
}
```

**문제점**:
1. **스레드 생성/삭제 오버헤드**: 시스템 콜, 메모리 할당 비용
2. **리소스 고갈**: 동시에 수천 개 스레드 생성 → 메모리/CPU 고갈
3. **컨텍스트 스위칭 폭증**: 스레드가 많을수록 오버헤드 증가
4. **동시성 제어 어려움**: 얼마나 많은 스레드가 실행 중인지 파악 불가
5. **캐시 지역성 저하**: 스레드 생성/삭제로 캐시 효율 저하

---

## 🏗️ 해결 방법: Thread Pool

### 아키텍처

```
클라이언트 요청
    ↓
┌─────────────────────────┐
│     작업 큐 (Queue)      │
│  [Task1][Task2][Task3]  │ ← submit(task)
└─────────────────────────┘
    │         │         │
    ↓         ↓         ↓
┌────────┐ ┌────────┐ ┌────────┐
│Worker 1│ │Worker 2│ │Worker 3│ ← 고정된 N개 워커 스레드
│(Thread)│ │(Thread)│ │(Thread)│
└────────┘ └────────┘ └────────┘
    ↓         ↓         ↓
  Task1     Task2     Task3  ← 작업 실행


생명주기:
1. 초기화: N개 워커 스레드 생성
2. 대기: 워커들이 작업 큐 대기
3. 제출: 클라이언트가 작업 제출
4. 실행: 워커가 작업 가져와서 실행
5. 반복: 작업 완료 후 다시 대기 상태
6. 종료: 남은 작업 완료 후 스레드 종료
```

---

## 💻 기본 구현

### 1. 간단한 Thread Pool

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
    std::vector<std::thread> workers_;           // 워커 스레드들
    std::queue<std::function<void()>> tasks_;    // 작업 큐

    std::mutex mutex_;
    std::condition_variable condition_;
    bool stop_ = false;

    // 워커 스레드 함수
    void worker_thread() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(mutex_);

                // 작업이 있거나 종료 신호가 올 때까지 대기
                condition_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });

                // 종료 신호 && 큐가 비어있으면 종료
                if (stop_ && tasks_.empty()) {
                    return;
                }

                // 작업 가져오기
                task = std::move(tasks_.front());
                tasks_.pop();
            }

            // 락 해제 후 작업 실행
            task();
        }
    }

public:
    // 생성자: num_threads 개의 워커 스레드 생성
    explicit ThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                worker_thread();
            });
        }
    }

    // 소멸자: 모든 작업 완료 후 종료
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

    // 작업 제출 (Future 지원)
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

            // 종료 후에는 작업 제출 불가
            if (stop_) {
                throw std::runtime_error("Cannot submit to stopped ThreadPool");
            }

            tasks_.emplace([task]() { (*task)(); });
        }

        condition_.notify_one();
        return result;
    }

    // 현재 대기 중인 작업 수
    size_t pending_tasks() const {
        std::unique_lock<std::mutex> lock(mutex_);
        return tasks_.size();
    }
};
```

---

## 🚀 사용 예시

### 예시 1: 기본 사용

```cpp
#include <iostream>
#include <chrono>

int compute(int x) {
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    return x * x;
}

int main() {
    // 4개 워커 스레드로 풀 생성
    ThreadPool pool(4);

    // 작업 제출 및 Future 받기
    std::vector<std::future<int>> results;

    for (int i = 0; i < 10; ++i) {
        results.push_back(pool.submit(compute, i));
    }

    // 결과 수집
    for (int i = 0; i < results.size(); ++i) {
        std::cout << "Result " << i << ": " << results[i].get() << "\n";
    }

    return 0;
}
```

**출력**:
```
Result 0: 0
Result 1: 1
Result 2: 4
Result 3: 9
...
Result 9: 81
```

### 예시 2: 병렬 파일 처리

```cpp
void process_file(const std::string& filename) {
    std::cout << "Processing " << filename << " on thread "
              << std::this_thread::get_id() << "\n";
    // 파일 처리 로직
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
}

int main() {
    ThreadPool pool(std::thread::hardware_concurrency());

    std::vector<std::string> files = {
        "file1.txt", "file2.txt", "file3.txt", "file4.txt",
        "file5.txt", "file6.txt", "file7.txt", "file8.txt"
    };

    std::vector<std::future<void>> futures;
    for (const auto& file : files) {
        futures.push_back(pool.submit(process_file, file));
    }

    // 모든 파일 처리 완료 대기
    for (auto& future : futures) {
        future.wait();
    }

    std::cout << "All files processed!\n";
    return 0;
}
```

---

## 🔧 고급 기능

### 1. Priority Thread Pool (우선순위 큐)

```cpp
class PriorityThreadPool {
private:
    struct Task {
        int priority;
        std::function<void()> func;

        bool operator<(const Task& other) const {
            return priority < other.priority;  // 높은 우선순위가 먼저
        }
    };

    std::vector<std::thread> workers_;
    std::priority_queue<Task> tasks_;
    std::mutex mutex_;
    std::condition_variable condition_;
    bool stop_ = false;

public:
    explicit PriorityThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    Task task;
                    {
                        std::unique_lock<std::mutex> lock(mutex_);
                        condition_.wait(lock, [this] {
                            return stop_ || !tasks_.empty();
                        });

                        if (stop_ && tasks_.empty()) return;

                        task = tasks_.top();
                        tasks_.pop();
                    }
                    task.func();
                }
            });
        }
    }

    template<typename F>
    void submit(int priority, F&& func) {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            if (stop_) throw std::runtime_error("Pool stopped");

            tasks_.push(Task{priority, std::forward<F>(func)});
        }
        condition_.notify_one();
    }

    ~PriorityThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        condition_.notify_all();
        for (auto& worker : workers_) {
            if (worker.joinable()) worker.join();
        }
    }
};
```

**사용 예시**:
```cpp
PriorityThreadPool pool(4);

pool.submit(1, []() { std::cout << "Low priority\n"; });
pool.submit(10, []() { std::cout << "High priority\n"; });  // 먼저 실행됨
pool.submit(5, []() { std::cout << "Medium priority\n"; });
```

### 2. Dynamic Thread Pool (동적 크기 조정)

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
    std::atomic<size_t> idle_threads_{0};

public:
    DynamicThreadPool(size_t min_threads, size_t max_threads)
        : min_threads_(min_threads), max_threads_(max_threads) {

        for (size_t i = 0; i < min_threads_; ++i) {
            add_worker();
        }
    }

    void add_worker() {
        workers_.emplace_back([this] {
            while (true) {
                std::function<void()> task;
                {
                    std::unique_lock<std::mutex> lock(mutex_);
                    idle_threads_++;

                    condition_.wait(lock, [this] {
                        return stop_ || !tasks_.empty();
                    });

                    idle_threads_--;

                    if (stop_ && tasks_.empty()) return;

                    task = std::move(tasks_.front());
                    tasks_.pop();
                }
                task();
            }
        });
    }

    template<typename F>
    void submit(F&& func) {
        {
            std::unique_lock<std::mutex> lock(mutex_);

            // 작업이 많고 유휴 스레드가 없으면 스레드 추가
            if (idle_threads_ == 0 && workers_.size() < max_threads_) {
                add_worker();
            }

            tasks_.emplace(std::forward<F>(func));
        }
        condition_.notify_one();
    }

    ~DynamicThreadPool() {
        {
            std::unique_lock<std::mutex> lock(mutex_);
            stop_ = true;
        }
        condition_.notify_all();
        for (auto& worker : workers_) {
            if (worker.joinable()) worker.join();
        }
    }
};
```

### 3. Work Stealing Thread Pool

각 워커가 자신의 큐를 가지고, 작업이 없으면 다른 워커의 큐에서 "훔쳐" 옵니다.

```cpp
class WorkStealingThreadPool {
private:
    struct WorkerThread {
        std::deque<std::function<void()>> local_queue;
        std::mutex mutex;
    };

    std::vector<std::unique_ptr<WorkerThread>> workers_;
    std::vector<std::thread> threads_;
    std::atomic<bool> stop_{false};

    // 다른 워커의 큐에서 작업 훔치기
    bool try_steal(size_t worker_id, std::function<void()>& task) {
        for (size_t i = 0; i < workers_.size(); ++i) {
            if (i == worker_id) continue;

            std::unique_lock<std::mutex> lock(workers_[i]->mutex, std::try_to_lock);
            if (lock.owns_lock() && !workers_[i]->local_queue.empty()) {
                task = std::move(workers_[i]->local_queue.back());
                workers_[i]->local_queue.pop_back();
                return true;
            }
        }
        return false;
    }

    void worker_thread(size_t id) {
        while (!stop_) {
            std::function<void()> task;

            // 자신의 큐에서 작업 가져오기
            {
                std::unique_lock<std::mutex> lock(workers_[id]->mutex);
                if (!workers_[id]->local_queue.empty()) {
                    task = std::move(workers_[id]->local_queue.front());
                    workers_[id]->local_queue.pop_front();
                }
            }

            // 자신의 큐가 비어있으면 다른 워커에서 훔치기
            if (!task && !try_steal(id, task)) {
                std::this_thread::yield();
                continue;
            }

            if (task) {
                task();
            }
        }
    }

public:
    explicit WorkStealingThreadPool(size_t num_threads) {
        for (size_t i = 0; i < num_threads; ++i) {
            workers_.push_back(std::make_unique<WorkerThread>());
            threads_.emplace_back([this, i] { worker_thread(i); });
        }
    }

    void submit(std::function<void()> task) {
        // Round-robin으로 워커 선택
        static std::atomic<size_t> counter{0};
        size_t worker_id = counter++ % workers_.size();

        std::unique_lock<std::mutex> lock(workers_[worker_id]->mutex);
        workers_[worker_id]->local_queue.push_back(std::move(task));
    }

    ~WorkStealingThreadPool() {
        stop_ = true;
        for (auto& thread : threads_) {
            if (thread.joinable()) thread.join();
        }
    }
};
```

---

## 📊 성능 고려사항

### 최적 스레드 수 결정

| 작업 유형 | 최적 스레드 수 | 이유 |
|----------|---------------|------|
| **CPU-bound** | `코어 수` | CPU 활용 극대화 |
| **I/O-bound** | `코어 수 × (1 + 대기시간/CPU시간)` | I/O 대기 중에도 작업 처리 |
| **Mixed** | `코어 수 × 1.5 ~ 2` | 경험적 값 |

**예시 계산**:
```cpp
// CPU-bound
size_t num_threads = std::thread::hardware_concurrency();  // 예: 8

// I/O-bound (80% I/O 대기)
// 대기시간/CPU시간 = 80/20 = 4
size_t num_threads = std::thread::hardware_concurrency() * 5;  // 40
```

### 작업 큐 크기 제한

무제한 큐는 메모리 고갈 위험:

```cpp
class BoundedThreadPool {
private:
    std::queue<std::function<void()>> tasks_;
    size_t max_queue_size_;
    std::condition_variable producer_cv_;  // 생산자 대기용

public:
    BoundedThreadPool(size_t num_threads, size_t max_queue_size)
        : max_queue_size_(max_queue_size) {
        // ... 워커 생성
    }

    template<typename F>
    void submit(F&& func) {
        std::unique_lock<std::mutex> lock(mutex_);

        // 큐가 가득 차면 대기
        producer_cv_.wait(lock, [this] {
            return tasks_.size() < max_queue_size_ || stop_;
        });

        if (stop_) throw std::runtime_error("Pool stopped");

        tasks_.emplace(std::forward<F>(func));
        condition_.notify_one();
    }
};
```

---

## ⚠️ 주의사항

### 1. Deadlock 위험

```cpp
// ❌ 위험: 풀 내부에서 다른 작업 대기
pool.submit([]() {
    auto future = pool.submit(another_task);  // 데드락 가능!
    future.wait();
});
```

**해결**:
- 중첩 작업 금지
- 별도의 풀 사용
- 충분히 큰 풀 크기

### 2. 예외 처리

```cpp
// ✅ 예외 안전한 워커
void worker_thread() {
    while (true) {
        std::function<void()> task;
        // ... 작업 가져오기 ...

        try {
            task();
        } catch (const std::exception& e) {
            std::cerr << "Task exception: " << e.what() << "\n";
            // 로깅, 재시도 등
        } catch (...) {
            std::cerr << "Unknown task exception\n";
        }
    }
}
```

### 3. 리소스 누수

```cpp
// ❌ 위험: 풀이 파괴되지 않으면 스레드 누수
{
    ThreadPool* pool = new ThreadPool(4);
    // ... 사용 ...
    // delete 잊어버림!
}

// ✅ RAII 사용
{
    ThreadPool pool(4);
    // 스코프 종료 시 자동 정리
}
```

---

## 🎯 실전 사용 사례

### 1. 웹 서버 요청 처리

```cpp
class WebServer {
    ThreadPool pool_;

public:
    WebServer() : pool_(std::thread::hardware_concurrency() * 2) {}

    void handle_connection(Socket socket) {
        pool_.submit([socket]() {
            auto request = read_request(socket);
            auto response = process_request(request);
            send_response(socket, response);
            socket.close();
        });
    }
};
```

### 2. 이미지 배치 처리

```cpp
void process_images(const std::vector<std::string>& images) {
    ThreadPool pool(4);
    std::vector<std::future<void>> futures;

    for (const auto& img : images) {
        futures.push_back(pool.submit([&img]() {
            auto image = load_image(img);
            resize(image, 800, 600);
            apply_filter(image);
            save_image(image, "processed_" + img);
        }));
    }

    // 모든 이미지 처리 완료 대기
    for (auto& f : futures) {
        f.wait();
    }
}
```

### 3. 데이터베이스 배치 업데이트

```cpp
void batch_update(const std::vector<Record>& records) {
    ThreadPool pool(8);

    const size_t batch_size = 100;
    std::vector<std::future<void>> futures;

    for (size_t i = 0; i < records.size(); i += batch_size) {
        size_t end = std::min(i + batch_size, records.size());

        futures.push_back(pool.submit([&records, i, end]() {
            auto conn = db_connection_pool.get();
            for (size_t j = i; j < end; ++j) {
                conn->update(records[j]);
            }
        }));
    }

    for (auto& f : futures) {
        f.wait();
    }
}
```

---

## 📚 표준 라이브러리 Thread Pool

### C++23: std::execution

```cpp
#include <execution>  // C++23
#include <algorithm>
#include <vector>

// 자동으로 thread pool 사용
std::vector<int> data(1000);
std::transform(std::execution::par,  // 병렬 실행
               data.begin(), data.end(), data.begin(),
               [](int x) { return x * 2; });
```

### 기타 라이브러리

| 라이브러리 | 설명 |
|-----------|------|
| **Boost.Asio** | `io_context` + `thread_pool` |
| **Intel TBB** | `task_scheduler_init`, `parallel_for` |
| **std::async** | 단순한 비동기 실행 (내부적으로 thread pool) |

---

## 🔗 관련 문서

- [Producer-Consumer](./01-producer-consumer.md) - 작업 큐 패턴
- [Future/Promise](./05-future-promise.md) - 비동기 결과 처리
- [컨텍스트 스위칭](../01-fundamentals/04-context-switching.md) - 스레드 수 최적화

---

## 📖 참고 자료

- "C++ Concurrency in Action" - Anthony Williams (Chapter 9: Advanced Thread Management)
- [Boost.Asio Thread Pool](https://www.boost.org/doc/libs/1_82_0/doc/html/boost_asio/overview/core/basics.html)

---

*Thread Pool은 멀티스레드 프로그래밍에서 가장 실용적이고 널리 사용되는 패턴입니다!*
