# 기아 상태 (Starvation)

## 기아 상태란 무엇인가?

**기아 상태(Starvation)**는 스레드가 진행하는 데 필요한 자원에 대한 접근을 영구적으로 거부당할 때 발생합니다. 모든 스레드가 막힌 교착 상태와 달리, 기아 상태에서는 일부 스레드는 진행하지만 다른 스레드는 무한정 지연됩니다. 굶주린 스레드는 결국 자원을 얻을 수 있지만 대기 시간이 무제한이고 예측할 수 없습니다.

### 공식적 정의

스레드는 다음과 같은 경우 기아 상태를 겪습니다:
1. 실행할 준비가 되어 있고 자원이 필요함
2. 다른 스레드가 지속적으로 해당 자원을 획득함
3. 스레드가 진행 없이 무한정 대기함
4. 시스템 전체는 진행함 (교착 상태와 달리)

## 시각적 표현

```
┌──────────────────────────────────────────────────────┐
│                    Resource Access                    │
├──────────────────────────────────────────────────────┤
│ Time ────────────────────────────────────────────▶   │
│                                                       │
│ Thread 1 (High):  [███][███][███][███][███][███]    │
│ Thread 2 (High):     [███][███][███][███][███]      │
│ Thread 3 (Low):                                  ⏳   │
│                   ↑                                   │
│              STARVING THREAD                          │
│         (waiting but never served)                    │
└──────────────────────────────────────────────────────┘
```

### 기아 상태 vs 기타 문제

```
┌───────────────┬──────────┬──────────┬────────────┬──────────┐
│   Problem     │  Blocked │ Progress │   Cause    │ Severity │
├───────────────┼──────────┼──────────┼────────────┼──────────┤
│ Deadlock      │   All    │   None   │  Circular  │ Critical │
│ Livelock      │   None   │   None   │  Collision │   High   │
│ Starvation    │   Some   │  Partial │  Unfair    │  Medium  │
└───────────────┴──────────┴──────────┴────────────┴──────────┘
```

## 기아 상태의 일반적인 원인

### 1. 우선순위 기반 스케줄링

높은 우선순위 스레드가 항상 낮은 우선순위 스레드를 선점합니다.

```cpp
#include <thread>
#include <mutex>
#include <iostream>

std::mutex resource;

void high_priority_thread() {
    // Note: C++ std::thread doesn't provide cross-platform priority setting
    // Platform-specific APIs would be needed (e.g., pthread_setschedparam on POSIX)

    while (true) {
        std::lock_guard<std::mutex> lock(resource);
        std::cout << "High priority: Working\n";
        // Do work...
    }
}

void low_priority_thread() {
    // Note: C++ std::thread doesn't provide cross-platform priority setting
    // Platform-specific APIs would be needed

    while (true) {
        std::lock_guard<std::mutex> lock(resource);
        std::cout << "Low priority: Working\n";  // MAY NEVER PRINT
        // Do work...
    }
}

// Low priority thread can STARVE if high priority runs continuously
```

### 2. 불공정한 락 구현

일부 락 구현은 공정성을 보장하지 않습니다.

```cpp
#include <atomic>

// Unfair mutex implementation (simplified)
class UnfairMutex {
private:
    std::atomic<int> locked{0};
    // No queue - threads race to acquire

public:
    void lock() {
        // Spin until successful
        while (true) {
            int expected = 0;
            if (locked.compare_exchange_weak(expected, 1)) {
                return;  // Acquired
            }
            // Some threads might retry faster than others!
            // Fast threads can starve slow ones
        }
    }
};

// Thread with faster CPU core might always win
// Thread with slower core might STARVE
```

### 3. 독자-저자 문제

독자가 계속 도착하면 저자가 굶주릴 수 있습니다.

```cpp
#include <mutex>
#include <thread>

struct RWLock {
    std::mutex mutex;
    int readers;
};

RWLock rwlock = {std::mutex(), 0};

void read_lock(RWLock& lock) {
    std::lock_guard<std::mutex> lk(lock.mutex);
    lock.readers++;
}

void read_unlock(RWLock& lock) {
    std::lock_guard<std::mutex> lk(lock.mutex);
    lock.readers--;
}

void write_lock(RWLock& lock) {
    lock.mutex.lock();

    // Wait for all readers to finish
    while (lock.readers > 0) {
        lock.mutex.unlock();
        std::this_thread::yield();
        lock.mutex.lock();
    }

    // Now have write access
}

// PROBLEM: If readers keep arriving, writer STARVES
```

**타임라인:**
```
Time  Readers  Writer State
----  -------  ------------
  1     2      Waiting (readers = 2)
  2     3      Waiting (new reader arrived!)
  3     2      Waiting (one left, one joined)
  4     4      Waiting (more readers!)
  5     3      Still waiting...
  ...   ...    STARVING
```

### 4. 불공정한 세마포어를 사용한 생산자-소비자

```cpp
#include <semaphore>
#include <thread>

constexpr int BUFFER_SIZE = 10;

std::counting_semaphore<BUFFER_SIZE> empty{BUFFER_SIZE};  // Count of empty slots
std::counting_semaphore<BUFFER_SIZE> full{0};   // Count of full slots

void producer() {
    while (true) {
        empty.acquire();  // Wait for empty slot

        // Produce item
        produce_item();

        full.release();   // Signal item available
    }
}

void consumer() {
    while (true) {
        full.acquire();   // Wait for item

        // Consume item
        consume_item();

        empty.release();  // Signal slot empty
    }
}

// If many fast producers and one slow consumer,
// slow consumer might STARVE
```

## 카테고리별 예제

### 예제 1: 스레드 풀 기아 상태

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <array>
#include <iostream>

constexpr int NUM_WORKERS = 4;
constexpr int QUEUE_SIZE = 100;

struct Task {
    std::function<void()> function;
    int priority;  // Higher = more important
};

struct ThreadPool {
    std::array<Task, QUEUE_SIZE> queue;
    int size = 0;
    std::mutex mutex;
    std::condition_variable cond;
};

ThreadPool pool;

void enqueue_task(std::function<void()> func, int priority) {
    std::lock_guard<std::mutex> lock(pool.mutex);

    // Insert by priority (higher priority first)
    int i = pool.size;
    while (i > 0 && pool.queue[i-1].priority < priority) {
        pool.queue[i] = pool.queue[i-1];
        i--;
    }

    pool.queue[i].function = func;
    pool.queue[i].priority = priority;
    pool.size++;

    pool.cond.notify_one();
}

void worker() {
    while (true) {
        std::unique_lock<std::mutex> lock(pool.mutex);

        pool.cond.wait(lock, []{ return pool.size > 0; });

        // Take highest priority task
        Task task = pool.queue[0];
        pool.size--;

        // Shift queue
        for (int i = 0; i < pool.size; i++) {
            pool.queue[i] = pool.queue[i+1];
        }

        lock.unlock();

        // Execute task
        task.function();
    }
}

// PROBLEM: Low priority tasks can STARVE if high priority
// tasks keep arriving
```

**시각화:**
```
Queue State (priority-ordered):
[9][9][9][8][8][7][7][7][6][5] ← High priority kept arriving
                               [2] ← Low priority task STARVING
```

### 예제 2: 디스크 I/O 스케줄러 기아 상태

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int track;      // Disk track number
    int timestamp;  // When request arrived
} IORequest;

// SCAN (Elevator) algorithm - can cause starvation
void scan_schedule(IORequest* requests, int count, int current_track) {
    int direction = 1;  // 1 = up, -1 = down

    while (1) {
        int served = 0;

        // Serve requests in current direction
        for (int i = 0; i < count; i++) {
            if (requests[i].track >= current_track && direction == 1) {
                printf("Serving track %d (age: %d)\n",
                       requests[i].track,
                       get_age(requests[i].timestamp));
                current_track = requests[i].track;
                served++;
            }
        }

        if (served == 0) {
            direction = -direction;  // Reverse direction
        }

        // PROBLEM: Requests at far end can STARVE
        // if new requests keep arriving on current side
    }
}
```

### 예제 3: 네트워크 패킷 처리

```c
#include <stdio.h>
#include <stdint.h>

typedef struct {
    uint8_t priority;
    uint32_t data;
    uint64_t arrival_time;
} Packet;

#define QUEUE_SIZE 1000
Packet queue[QUEUE_SIZE];
int queue_size = 0;

void process_packets() {
    while (1) {
        if (queue_size == 0) continue;

        // Always process highest priority first
        int highest_idx = 0;
        for (int i = 1; i < queue_size; i++) {
            if (queue[i].priority > queue[highest_idx].priority) {
                highest_idx = i;
            }
        }

        Packet p = queue[highest_idx];

        // Check for starvation
        uint64_t wait_time = current_time() - p.arrival_time;
        if (wait_time > 10000) {  // 10 seconds
            printf("WARNING: Packet starved for %llu ms\n", wait_time);
        }

        process_packet(&p);

        // Remove from queue
        queue[highest_idx] = queue[--queue_size];
    }
}

// Low priority packets STARVE if high priority keep arriving
```

## 해결책 및 예방

### 해결책 1: 공정한 뮤텍스 (FIFO 순서)

```cpp
#include <mutex>
#include <condition_variable>

struct WaitNode {
    std::condition_variable cond;
    bool ready;
    WaitNode* next;
};

class FairMutex {
private:
    std::mutex mutex;
    WaitNode* head = nullptr;
    WaitNode* tail = nullptr;
    bool locked = false;

public:
    void lock() {
        WaitNode node;
        node.ready = false;
        node.next = nullptr;

        std::unique_lock<std::mutex> lk(mutex);

        if (!locked) {
            locked = true;
            return;  // Got lock immediately
        }

        // Add to wait queue
        if (tail) {
            tail->next = &node;
        } else {
            head = &node;
        }
        tail = &node;

        // Wait for our turn
        while (!node.ready) {
            node.cond.wait(lk);
        }
    }

    void unlock() {
        std::lock_guard<std::mutex> lk(mutex);

        if (head) {
            // Wake next waiter
            head->ready = true;
            head->cond.notify_one();
            head = head->next;
            if (!head) {
                tail = nullptr;
            }
        } else {
            locked = false;
        }
    }
};

// FIFO ordering prevents starvation
```

### 해결책 2: 공정한 독자-저자 락

```cpp
#include <mutex>
#include <condition_variable>

class FairRWLock {
private:
    std::mutex mutex;
    std::condition_variable readers_cond;
    std::condition_variable writers_cond;
    int readers = 0;
    int writers = 0;
    int waiting_writers = 0;

public:
    void read_lock() {
        std::unique_lock<std::mutex> lock(mutex);

        // Wait if there's a writer or waiting writers
        readers_cond.wait(lock, [this] {
            return writers == 0 && waiting_writers == 0;
        });

        readers++;
    }

    void read_unlock() {
        std::lock_guard<std::mutex> lock(mutex);
        readers--;

        if (readers == 0 && waiting_writers > 0) {
            // Wake a waiting writer
            writers_cond.notify_one();
        }
    }

    void write_lock() {
        std::unique_lock<std::mutex> lock(mutex);
        waiting_writers++;

        // Wait for readers and writers to finish
        writers_cond.wait(lock, [this] {
            return readers == 0 && writers == 0;
        });

        waiting_writers--;
        writers++;
    }

    void write_unlock() {
        std::lock_guard<std::mutex> lock(mutex);
        writers--;

        if (waiting_writers > 0) {
            // Prefer waiting writers
            writers_cond.notify_one();
        } else {
            // Wake all waiting readers
            readers_cond.notify_all();
        }
    }
};

// Writers won't starve - they're preferred after current readers
```

### 해결책 3: 에이징 우선순위

시간이 지남에 따라 대기 중인 스레드의 우선순위를 증가시킵니다.

```cpp
#include <chrono>
#include <functional>
#include <vector>

struct AgingTask {
    std::function<void()> function;
    int base_priority;
    std::chrono::time_point<std::chrono::steady_clock> enqueue_time;
};

int effective_priority(const AgingTask& task) {
    auto now = std::chrono::steady_clock::now();
    auto age = std::chrono::duration_cast<std::chrono::seconds>(now - task.enqueue_time);
    // Increase priority by 1 every 10 seconds
    int age_bonus = age.count() / 10;
    return task.base_priority + age_bonus;
}

AgingTask* get_next_task(std::vector<AgingTask>& queue) {
    if (queue.empty()) return nullptr;

    int best_idx = 0;
    int best_priority = effective_priority(queue[0]);

    for (size_t i = 1; i < queue.size(); i++) {
        int priority = effective_priority(queue[i]);
        if (priority > best_priority) {
            best_priority = priority;
            best_idx = i;
        }
    }

    return &queue[best_idx];
}

// Old low-priority tasks eventually become high priority
// Prevents indefinite starvation
```

### 해결책 4: 라운드 로빈 스케줄링

각 스레드에 타임 슬라이스를 부여합니다.

```cpp
#include <thread>
#include <vector>
#include <chrono>

constexpr int NUM_THREADS = 5;
constexpr int TIME_SLICE_MS = 100;

std::vector<std::thread> threads;
int current_thread = 0;

void setup_round_robin() {
    // Note: C++ standard library doesn't provide direct thread
    // suspension/resumption. This requires platform-specific code.
    // On POSIX systems, you would use pthread_kill with SIGSTOP/SIGCONT
    // On Windows, you would use SuspendThread/ResumeThread
    //
    // This is a conceptual example - actual implementation would need
    // platform-specific code or a cooperative scheduling approach
}

// All threads get equal CPU time - no starvation
// Note: Preemptive scheduling requires OS/platform-specific APIs
```

### 해결책 5: 2단계 피드백 큐

```c
#define NUM_QUEUES 3

typedef struct {
    Task queues[NUM_QUEUES][100];
    int sizes[NUM_QUEUES];
    int execution_counts[1000];  // Track per-thread
} FeedbackQueue;

FeedbackQueue fbq = {{{0}}, {0}, {0}};

void enqueue_with_feedback(int thread_id, Task task) {
    // New tasks start at highest priority queue
    int queue_level = 0;

    // Demote if executed too many times
    int exec_count = fbq.execution_counts[thread_id];
    if (exec_count > 10) queue_level = 2;      // Low priority
    else if (exec_count > 3) queue_level = 1;  // Medium priority

    fbq.queues[queue_level][fbq.sizes[queue_level]++] = task;
}

Task* get_next_with_feedback() {
    // Service higher priority queues first
    for (int level = 0; level < NUM_QUEUES; level++) {
        if (fbq.sizes[level] > 0) {
            Task* task = &fbq.queues[level][0];

            // Remove from queue
            for (int i = 0; i < fbq.sizes[level] - 1; i++) {
                fbq.queues[level][i] = fbq.queues[level][i + 1];
            }
            fbq.sizes[level]--;

            return task;
        }
    }
    return NULL;
}

// Even low-priority tasks get served when high queue is empty
```

## 탐지 전략

### 1. 대기 시간 모니터링

```cpp
#include <chrono>
#include <iostream>
#include <thread>
#include <vector>
#include <string>

constexpr int STARVATION_THRESHOLD_MS = 5000;

struct WaitInfo {
    std::thread::id thread_id;
    std::chrono::time_point<std::chrono::steady_clock> wait_start;
    std::string resource_name;
};

std::vector<WaitInfo> waiting_threads;

void monitor_wait_times() {
    auto now = std::chrono::steady_clock::now();

    for (const auto& info : waiting_threads) {
        auto wait_time = std::chrono::duration_cast<std::chrono::milliseconds>(
            now - info.wait_start);

        if (wait_time.count() > STARVATION_THRESHOLD_MS) {
            std::cout << "STARVATION ALERT: Thread " << info.thread_id
                      << " waiting " << wait_time.count() / 1000
                      << " seconds for " << info.resource_name << "\n";
        }
    }
}
```

### 2. 공정성 메트릭

```cpp
#include <vector>
#include <iostream>

struct ThreadStats {
    int thread_id;
    int acquisitions;
    long total_hold_time;
    long total_wait_time;
};

void calculate_fairness(const std::vector<ThreadStats>& stats) {
    long total_acquisitions = 0;

    for (const auto& stat : stats) {
        total_acquisitions += stat.acquisitions;
    }
    long avg_acquisitions = total_acquisitions / stats.size();

    std::cout << "Fairness Analysis:\n";
    for (const auto& stat : stats) {
        double deviation = static_cast<double>(stat.acquisitions - avg_acquisitions)
                          / avg_acquisitions * 100;

        std::cout << "Thread " << stat.thread_id << ": " << stat.acquisitions
                  << " acquisitions (" << deviation << "% from average)\n";

        if (deviation < -50) {
            std::cout << "  WARNING: Potential starvation!\n";
        }
    }
}
```

### 3. 큐 길이 추적

```c
void track_queue_length(int queue_length, int thread_id) {
    static int max_queue_length[100] = {0};

    if (queue_length > max_queue_length[thread_id]) {
        max_queue_length[thread_id] = queue_length;
    }

    if (queue_length > 50) {
        printf("WARNING: Thread %d in long queue (%d deep)\n",
               thread_id, queue_length);
    }
}
```

## 공정성 개념

### 강한 공정성

접근을 원하는 모든 스레드는 결국 얻을 것입니다.

```c
// Example: FIFO mutex (shown earlier)
// Guarantees: If thread requests lock, it WILL get it
```

### 약한 공정성

스레드가 계속해서 접근을 원하면 결국 얻을 것입니다.

```c
// Example: Simple mutex with no guarantees
// Only ensures: continuous requests eventually succeed
```

### 공정성 없음

누가 언제 접근할지에 대한 보장이 없습니다.

```c
// Example: Spinlock without queue
while (!atomic_compare_exchange(&lock, &expected, 1)) {
    // Any thread might win - no fairness
}
```

### 공정성 비교

```
┌─────────────────┬──────────────┬───────────────┬──────────┐
│   Mechanism     │   Fairness   │   Overhead    │ Starvation│
├─────────────────┼──────────────┼───────────────┼──────────┤
│ Spinlock        │     None     │      Low      │   Possible│
│ Basic Mutex     │     Weak     │     Medium    │   Possible│
│ FIFO Mutex      │    Strong    │      High     │     No    │
│ Priority Mutex  │     None     │     Medium    │   Likely  │
│ RR Scheduling   │    Strong    │     Medium    │     No    │
└─────────────────┴──────────────┴───────────────┴──────────┘
```

## 모범 사례

### 해야 할 것:
- ✓ 공정한 동기화 프리미티브 사용
- ✓ 대기 시간 모니터링 및 기아 상태 탐지
- ✓ 우선순위 시스템을 위한 에이징 구현
- ✓ 우선순위 범위 제한
- ✓ 가능한 경우 FIFO 큐 사용
- ✓ 타임아웃 제한 설정
- ✓ 높은 부하 조건에서 테스트

### 하지 말아야 할 것:
- ✗ 무제한 우선순위 사용
- ✗ 항상 한 클래스의 스레드를 선호
- ✗ 대기 시간 메트릭 무시
- ✗ 검증 없이 공정성 가정
- ✗ 장기 실행 작업에 순수 우선순위 스케줄링 사용

## 요약

**기아 상태**는 다음과 같은 이유로 스레드가 영구적으로 자원을 거부당할 때 발생합니다:
- 불공정한 스케줄링
- 우선순위 체계
- 독자-저자 불균형
- 공정성 보장 부족

**주요 차이점:**
```
Deadlock:    No progress by anyone
Livelock:    Activity but no progress
Starvation:  Some progress, but not by everyone
```

**예방 전략:**
1. 공정한 락 (FIFO 순서)
2. 에이징 알고리즘
3. 제한된 대기
4. 라운드 로빈 스케줄링
5. 공정성 모니터링

## 연습 문제

### 연습 1: 기아 상태 탐지
스레드가 5초 이상 대기했을 때를 탐지하는 모니터링을 추가하십시오.

### 연습 2: 공정한 큐 구현
오래된 낮은 우선순위 항목이 결국 서비스되는 공정한 우선순위 큐를 만드십시오.

### 연습 3: 독자 기아 상태 수정
저자 기아 상태를 방지하도록 독자-저자 락을 수정하십시오.

## 추가 자료

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Modern Operating Systems" - Andrew Tanenbaum

## 다음 주제

[05-priority-inversion.md](./05-priority-inversion.md)로 계속하여 우선순위 역전에 대해 배우십시오.
