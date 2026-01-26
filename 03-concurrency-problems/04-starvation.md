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

```c
#include <pthread.h>
#include <stdio.h>
#include <sched.h>

pthread_mutex_t resource = PTHREAD_MUTEX_INITIALIZER;

void* high_priority_thread(void* arg) {
    // Set high priority
    struct sched_param param;
    param.sched_priority = 99;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("High priority: Working\n");
        // Do work...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

void* low_priority_thread(void* arg) {
    // Set low priority
    struct sched_param param;
    param.sched_priority = 1;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("Low priority: Working\n");  // MAY NEVER PRINT
        // Do work...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

// Low priority thread can STARVE if high priority runs continuously
```

### 2. 불공정한 락 구현

일부 락 구현은 공정성을 보장하지 않습니다.

```c
// Unfair mutex implementation (simplified)
typedef struct {
    atomic_int locked;
    // No queue - threads race to acquire
} UnfairMutex;

void unfair_lock(UnfairMutex* m) {
    // Spin until successful
    while (1) {
        int expected = 0;
        if (atomic_compare_exchange_weak(&m->locked, &expected, 1)) {
            return;  // Acquired
        }
        // Some threads might retry faster than others!
        // Fast threads can starve slow ones
    }
}

// Thread with faster CPU core might always win
// Thread with slower core might STARVE
```

### 3. 독자-저자 문제

독자가 계속 도착하면 저자가 굶주릴 수 있습니다.

```c
#include <pthread.h>
#include <stdio.h>

typedef struct {
    pthread_mutex_t mutex;
    int readers;
} RWLock;

RWLock rwlock = {PTHREAD_MUTEX_INITIALIZER, 0};

void read_lock(RWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->readers++;
    pthread_mutex_unlock(&lock->mutex);
}

void read_unlock(RWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->readers--;
    pthread_mutex_unlock(&lock->mutex);
}

void write_lock(RWLock* lock) {
    pthread_mutex_lock(&lock->mutex);

    // Wait for all readers to finish
    while (lock->readers > 0) {
        pthread_mutex_unlock(&lock->mutex);
        sched_yield();
        pthread_mutex_lock(&lock->mutex);
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

```c
#include <semaphore.h>
#include <pthread.h>

#define BUFFER_SIZE 10

sem_t empty;  // Count of empty slots
sem_t full;   // Count of full slots

void* producer(void* arg) {
    while (1) {
        sem_wait(&empty);  // Wait for empty slot

        // Produce item
        produce_item();

        sem_post(&full);   // Signal item available
    }
    return NULL;
}

void* consumer(void* arg) {
    while (1) {
        sem_wait(&full);   // Wait for item

        // Consume item
        consume_item();

        sem_post(&empty);  // Signal slot empty
    }
    return NULL;
}

// If many fast producers and one slow consumer,
// slow consumer might STARVE
```

## 카테고리별 예제

### 예제 1: 스레드 풀 기아 상태

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

#define NUM_WORKERS 4
#define QUEUE_SIZE 100

typedef struct {
    void (*function)(void*);
    void* arg;
    int priority;  // Higher = more important
} Task;

typedef struct {
    Task queue[QUEUE_SIZE];
    int size;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
} ThreadPool;

ThreadPool pool = {{}, 0, PTHREAD_MUTEX_INITIALIZER, PTHREAD_COND_INITIALIZER};

void enqueue_task(void (*func)(void*), void* arg, int priority) {
    pthread_mutex_lock(&pool.mutex);

    // Insert by priority (higher priority first)
    int i = pool.size;
    while (i > 0 && pool.queue[i-1].priority < priority) {
        pool.queue[i] = pool.queue[i-1];
        i--;
    }

    pool.queue[i].function = func;
    pool.queue[i].arg = arg;
    pool.queue[i].priority = priority;
    pool.size++;

    pthread_cond_signal(&pool.cond);
    pthread_mutex_unlock(&pool.mutex);
}

void* worker(void* arg) {
    while (1) {
        pthread_mutex_lock(&pool.mutex);

        while (pool.size == 0) {
            pthread_cond_wait(&pool.cond, &pool.mutex);
        }

        // Take highest priority task
        Task task = pool.queue[0];
        pool.size--;

        // Shift queue
        for (int i = 0; i < pool.size; i++) {
            pool.queue[i] = pool.queue[i+1];
        }

        pthread_mutex_unlock(&pool.mutex);

        // Execute task
        task.function(task.arg);
    }
    return NULL;
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

```c
#include <pthread.h>
#include <stdbool.h>

typedef struct WaitNode {
    pthread_cond_t cond;
    bool ready;
    struct WaitNode* next;
} WaitNode;

typedef struct {
    pthread_mutex_t mutex;
    WaitNode* head;
    WaitNode* tail;
    bool locked;
} FairMutex;

void fair_mutex_init(FairMutex* fm) {
    pthread_mutex_init(&fm->mutex, NULL);
    fm->head = fm->tail = NULL;
    fm->locked = false;
}

void fair_mutex_lock(FairMutex* fm) {
    WaitNode node;
    pthread_cond_init(&node.cond, NULL);
    node.ready = false;
    node.next = NULL;

    pthread_mutex_lock(&fm->mutex);

    if (!fm->locked) {
        fm->locked = true;
        pthread_mutex_unlock(&fm->mutex);
        return;  // Got lock immediately
    }

    // Add to wait queue
    if (fm->tail) {
        fm->tail->next = &node;
    } else {
        fm->head = &node;
    }
    fm->tail = &node;

    // Wait for our turn
    while (!node.ready) {
        pthread_cond_wait(&node.cond, &fm->mutex);
    }

    pthread_mutex_unlock(&fm->mutex);
    pthread_cond_destroy(&node.cond);
}

void fair_mutex_unlock(FairMutex* fm) {
    pthread_mutex_lock(&fm->mutex);

    if (fm->head) {
        // Wake next waiter
        fm->head->ready = true;
        pthread_cond_signal(&fm->head->cond);
        fm->head = fm->head->next;
        if (!fm->head) {
            fm->tail = NULL;
        }
    } else {
        fm->locked = false;
    }

    pthread_mutex_unlock(&fm->mutex);
}

// FIFO ordering prevents starvation
```

### 해결책 2: 공정한 독자-저자 락

```c
#include <pthread.h>
#include <stdbool.h>

typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t readers_cond;
    pthread_cond_t writers_cond;
    int readers;
    int writers;
    int waiting_writers;
} FairRWLock;

void fair_rwlock_init(FairRWLock* lock) {
    pthread_mutex_init(&lock->mutex, NULL);
    pthread_cond_init(&lock->readers_cond, NULL);
    pthread_cond_init(&lock->writers_cond, NULL);
    lock->readers = 0;
    lock->writers = 0;
    lock->waiting_writers = 0;
}

void fair_read_lock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);

    // Wait if there's a writer or waiting writers
    while (lock->writers > 0 || lock->waiting_writers > 0) {
        pthread_cond_wait(&lock->readers_cond, &lock->mutex);
    }

    lock->readers++;
    pthread_mutex_unlock(&lock->mutex);
}

void fair_read_unlock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->readers--;

    if (lock->readers == 0 && lock->waiting_writers > 0) {
        // Wake a waiting writer
        pthread_cond_signal(&lock->writers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

void fair_write_lock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->waiting_writers++;

    // Wait for readers and writers to finish
    while (lock->readers > 0 || lock->writers > 0) {
        pthread_cond_wait(&lock->writers_cond, &lock->mutex);
    }

    lock->waiting_writers--;
    lock->writers++;
    pthread_mutex_unlock(&lock->mutex);
}

void fair_write_unlock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->writers--;

    if (lock->waiting_writers > 0) {
        // Prefer waiting writers
        pthread_cond_signal(&lock->writers_cond);
    } else {
        // Wake all waiting readers
        pthread_cond_broadcast(&lock->readers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

// Writers won't starve - they're preferred after current readers
```

### 해결책 3: 에이징 우선순위

시간이 지남에 따라 대기 중인 스레드의 우선순위를 증가시킵니다.

```c
#include <time.h>
#include <pthread.h>

typedef struct {
    void (*function)(void*);
    void* arg;
    int base_priority;
    time_t enqueue_time;
} AgingTask;

int effective_priority(AgingTask* task) {
    time_t age = time(NULL) - task->enqueue_time;
    // Increase priority by 1 every 10 seconds
    int age_bonus = age / 10;
    return task->base_priority + age_bonus;
}

AgingTask* get_next_task(AgingTask* queue, int size) {
    int best_idx = 0;
    int best_priority = effective_priority(&queue[0]);

    for (int i = 1; i < size; i++) {
        int priority = effective_priority(&queue[i]);
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

```c
#include <pthread.h>
#include <signal.h>
#include <time.h>

#define NUM_THREADS 5
#define TIME_SLICE_MS 100

pthread_t threads[NUM_THREADS];
int current_thread = 0;

void switch_thread(int sig) {
    // Pause current thread
    pthread_kill(threads[current_thread], SIGSTOP);

    // Switch to next thread
    current_thread = (current_thread + 1) % NUM_THREADS;

    // Resume next thread
    pthread_kill(threads[current_thread], SIGCONT);
}

void setup_round_robin() {
    struct sigaction sa;
    sa.sa_handler = switch_thread;
    sigemptyset(&sa.sa_mask);
    sa.sa_flags = 0;
    sigaction(SIGALRM, &sa, NULL);

    struct itimerval timer;
    timer.it_value.tv_sec = 0;
    timer.it_value.tv_usec = TIME_SLICE_MS * 1000;
    timer.it_interval = timer.it_value;
    setitimer(ITIMER_REAL, &timer, NULL);
}

// All threads get equal CPU time - no starvation
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

```c
#include <time.h>
#include <stdio.h>

#define STARVATION_THRESHOLD_MS 5000

typedef struct {
    pthread_t thread_id;
    time_t wait_start;
    const char* resource_name;
} WaitInfo;

WaitInfo waiting_threads[100];
int num_waiting = 0;

void monitor_wait_times() {
    time_t now = time(NULL);

    for (int i = 0; i < num_waiting; i++) {
        time_t wait_time = now - waiting_threads[i].wait_start;

        if (wait_time > STARVATION_THRESHOLD_MS / 1000) {
            printf("STARVATION ALERT: Thread %lu waiting %ld seconds for %s\n",
                   waiting_threads[i].thread_id,
                   wait_time,
                   waiting_threads[i].resource_name);
        }
    }
}
```

### 2. 공정성 메트릭

```c
typedef struct {
    int thread_id;
    int acquisitions;
    long total_hold_time;
    long total_wait_time;
} ThreadStats;

void calculate_fairness(ThreadStats* stats, int num_threads) {
    long total_acquisitions = 0;
    long avg_acquisitions = 0;

    for (int i = 0; i < num_threads; i++) {
        total_acquisitions += stats[i].acquisitions;
    }
    avg_acquisitions = total_acquisitions / num_threads;

    printf("Fairness Analysis:\n");
    for (int i = 0; i < num_threads; i++) {
        double deviation = (double)(stats[i].acquisitions - avg_acquisitions)
                          / avg_acquisitions * 100;

        printf("Thread %d: %d acquisitions (%.1f%% from average)\n",
               stats[i].thread_id, stats[i].acquisitions, deviation);

        if (deviation < -50) {
            printf("  WARNING: Potential starvation!\n");
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
