# Starvation

## What is Starvation?

**Starvation** occurs when a thread is perpetually denied access to resources it needs to make progress. Unlike deadlock where all threads are stuck, in starvation some threads make progress while others are indefinitely delayed. The starved thread may eventually get the resource, but the wait time is unbounded and unpredictable.

### Formal Definition

A thread suffers from starvation when:
1. It is ready to execute and needs resources
2. Other threads continuously acquire those resources
3. The thread waits indefinitely without making progress
4. The system as a whole makes progress (unlike deadlock)

## Visual Representation

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

### Starvation vs Other Problems

```
┌───────────────┬──────────┬──────────┬────────────┬──────────┐
│   Problem     │  Blocked │ Progress │   Cause    │ Severity │
├───────────────┼──────────┼──────────┼────────────┼──────────┤
│ Deadlock      │   All    │   None   │  Circular  │ Critical │
│ Livelock      │   None   │   None   │  Collision │   High   │
│ Starvation    │   Some   │  Partial │  Unfair    │  Medium  │
└───────────────┴──────────┴──────────┴────────────┴──────────┘
```

## Common Causes of Starvation

### 1. Priority-Based Scheduling

High-priority threads always preempt low-priority threads.

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

### 2. Unfair Lock Implementation

Some lock implementations don't guarantee fairness.

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

### 3. Reader-Writer Problem

Writers can starve if readers keep arriving.

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

**Timeline:**
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

### 4. Producer-Consumer with Unfair Semaphore

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

## Examples by Category

### Example 1: Thread Pool Starvation

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

**Visualization:**
```
Queue State (priority-ordered):
[9][9][9][8][8][7][7][7][6][5] ← High priority kept arriving
                               [2] ← Low priority task STARVING
```

### Example 2: Disk I/O Scheduler Starvation

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

### Example 3: Network Packet Processing

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

## Solutions and Prevention

### Solution 1: Fair Mutex (FIFO Order)

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

### Solution 2: Fair Reader-Writer Lock

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

### Solution 3: Aging Priority

Increase priority of waiting threads over time.

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

### Solution 4: Round-Robin Scheduling

Give each thread a time slice.

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

### Solution 5: Two-Level Feedback Queue

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

## Internal Mechanisms

### Linux CFS 스케줄러의 공정성 보장

Linux의 Completely Fair Scheduler(CFS)는 vruntime을 통해 기아 상태를 방지합니다.

#### vruntime 계산

```c
// linux/kernel/sched/fair.c

/*
 * vruntime = 실제 실행 시간 × (NICE_0_LOAD / 스레드 weight)
 *
 * weight는 nice 값에 따라 결정:
 *   nice  0: weight = 1024 (기준)
 *   nice -1: weight = 1277 (25% 더 많은 CPU)
 *   nice +1: weight = 820  (25% 적은 CPU)
 */

static void update_curr(struct cfs_rq *cfs_rq) {
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    // 실제 실행 시간
    delta_exec = now - curr->exec_start;
    curr->exec_start = now;

    // 통계 업데이트
    curr->sum_exec_runtime += delta_exec;

    // vruntime 계산 (가중치 적용)
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    // 최소 vruntime 업데이트
    update_min_vruntime(cfs_rq);
}

static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se) {
    // delta × (NICE_0_LOAD / se->load.weight)
    if (unlikely(se->load.weight != NICE_0_LOAD))
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

    return delta;
}
```

#### 레드블랙 트리 기반 스케줄링

```
                    CFS 런큐 구조

              ┌─────────────────────┐
              │    Red-Black Tree   │
              │  (vruntime 정렬)    │
              └─────────┬───────────┘
                        │
             ┌──────────┴──────────┐
             │                     │
        ┌────┴────┐           ┌────┴────┐
        │  vr=100 │           │  vr=200 │
        │ (실행)  │           │         │
        └────┬────┘           └────┬────┘
             │                     │
        ┌────┴────┐           ┌────┴────┐
        │  vr=50  │           │  vr=150 │
        │ ← 다음! │           │         │
        └─────────┘           └─────────┘

  - 항상 가장 작은 vruntime (왼쪽 끝) 선택
  - 실행 시 vruntime 증가 → 트리 재정렬
  - 모든 태스크가 결국 가장 작은 vruntime을 가지게 됨
  - 기아 상태 불가능!
```

#### Aging 메커니즘

```c
// 새 태스크 또는 wakeup 시 vruntime 설정
static void place_entity(struct cfs_rq *cfs_rq, struct sched_entity *se,
                        int initial) {
    u64 vruntime = cfs_rq->min_vruntime;

    // 새 태스크: 약간의 보너스 (빠른 시작)
    if (initial)
        vruntime += sched_vslice(cfs_rq, se);

    // 오래 잠들었던 태스크: min_vruntime에 맞춤
    // (너무 작은 vruntime으로 인한 CPU 독점 방지)
    se->vruntime = max_vruntime(se->vruntime, vruntime);
}

// 잠들었다 깨어난 태스크의 vruntime 조정
static void task_waking_fair(struct task_struct *p) {
    struct sched_entity *se = &p->se;
    struct cfs_rq *cfs_rq = cfs_rq_of(se);

    // 잠든 동안의 시간을 보상하지 않음
    // 대신 현재 min_vruntime과 비교하여 적절한 위치에 삽입
    se->vruntime -= cfs_rq->min_vruntime;
}
```

### 실시간 스케줄러와 기아

```c
// SCHED_FIFO/SCHED_RR에서의 기아 문제와 해결책

// Linux의 RT throttling (기아 방지)
// /proc/sys/kernel/sched_rt_runtime_us (기본: 950000)
// /proc/sys/kernel/sched_rt_period_us  (기본: 1000000)

// 의미: RT 태스크는 1초 중 최대 0.95초만 실행
// 나머지 0.05초는 일반 태스크에게 보장

// 커널 구현 (간략화)
static void check_rt_throttle(struct rt_rq *rt_rq) {
    if (rt_rq->rt_time > rt_rq->rt_runtime) {
        // RT 태스크가 할당량 초과
        rt_rq->rt_throttled = 1;

        // 일반 태스크 스케줄링 허용
        resched_curr(rq);
    }
}

// 주기적으로 RT 시간 리셋
static enum hrtimer_restart sched_rt_period_timer(struct hrtimer *timer) {
    struct rt_rq *rt_rq = container_of(timer, struct rt_rq, rt_period_timer);

    // 새 주기 시작: 할당량 리셋
    rt_rq->rt_time = 0;
    rt_rq->rt_throttled = 0;

    // 다음 주기 타이머 설정
    hrtimer_forward_now(timer, rt_rq->rt_period);
    return HRTIMER_RESTART;
}
```

### PTHREAD 뮤텍스의 공정성

```c
// glibc의 PTHREAD_MUTEX_ADAPTIVE_NP 구현
// 짧은 대기는 spin, 긴 대기는 futex

#define MAX_SPIN_COUNT 100

int __pthread_mutex_lock(pthread_mutex_t *mutex) {
    int type = mutex->__data.__kind;

    // Adaptive mutex: spin 먼저 시도
    if (type == PTHREAD_MUTEX_ADAPTIVE_NP) {
        int spin_count = MAX_SPIN_COUNT;

        while (spin_count-- > 0) {
            if (lll_trylock(&mutex->__data.__lock) == 0) {
                return 0;  // spin 중 획득 성공
            }
            cpu_relax();
        }
    }

    // Spin 실패 또는 일반 mutex: futex 대기
    // 여기서 FIFO 보장은 futex 구현에 의존

    // FUTEX_WAIT_PRIVATE with FIFO semantics
    while (lll_cmpxchg(&mutex->__data.__lock, 0, 1) != 0) {
        // 대기자로 등록 (2 = contended)
        int oldval = atomic_exchange(&mutex->__data.__lock, 2);

        if (oldval != 0) {
            // futex 대기 (커널이 FIFO 순서로 깨움)
            futex_wait(&mutex->__data.__lock, 2);
        }
    }

    return 0;
}
```

### 읽기-쓰기 락에서의 공정성 구현

```c
// Writer 우선 RWLock의 내부 동작
typedef struct {
    atomic_int state;
    // 비트 레이아웃:
    // [31]: writer_active
    // [30]: writer_pending
    // [29:0]: reader_count

    futex_t writer_futex;
    futex_t reader_futex;
} fair_rwlock_t;

#define WRITER_ACTIVE  (1U << 31)
#define WRITER_PENDING (1U << 30)
#define READER_MASK    ((1U << 30) - 1)

void fair_rwlock_rdlock(fair_rwlock_t *lock) {
    int state;

    while (1) {
        state = atomic_load(&lock->state);

        // Writer가 대기 중이면 Reader 진입 차단 (기아 방지)
        if (state & (WRITER_ACTIVE | WRITER_PENDING)) {
            futex_wait(&lock->reader_futex, state);
            continue;
        }

        // Reader 카운트 증가 시도
        if (atomic_compare_exchange_weak(&lock->state, &state,
                                         state + 1)) {
            return;  // 성공
        }
    }
}

void fair_rwlock_wrlock(fair_rwlock_t *lock) {
    int state;

    // 1단계: Writer 대기 플래그 설정
    while (1) {
        state = atomic_load(&lock->state);

        if (atomic_compare_exchange_weak(&lock->state, &state,
                                         state | WRITER_PENDING)) {
            break;
        }
    }

    // 2단계: 모든 Reader 종료 대기
    while (1) {
        state = atomic_load(&lock->state);

        if ((state & READER_MASK) == 0 && !(state & WRITER_ACTIVE)) {
            // Reader 없고 다른 Writer 없음
            int new_state = (state & ~WRITER_PENDING) | WRITER_ACTIVE;
            if (atomic_compare_exchange_weak(&lock->state, &state,
                                             new_state)) {
                return;  // 성공
            }
        } else {
            futex_wait(&lock->writer_futex, state);
        }
    }
}
```

### 타임아웃 기반 기아 방지

```c
// 최대 대기 시간 보장

struct fair_resource {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    bool in_use;

    // 대기 큐 (FIFO)
    struct waiter *head;
    struct waiter *tail;
};

struct waiter {
    pthread_t thread;
    struct timespec deadline;
    struct waiter *next;
    bool ready;
};

int acquire_with_deadline(struct fair_resource *res,
                          int timeout_ms) {
    struct waiter w;
    w.thread = pthread_self();
    w.next = NULL;
    w.ready = false;

    // deadline 계산
    clock_gettime(CLOCK_REALTIME, &w.deadline);
    w.deadline.tv_sec += timeout_ms / 1000;
    w.deadline.tv_nsec += (timeout_ms % 1000) * 1000000;
    if (w.deadline.tv_nsec >= 1000000000) {
        w.deadline.tv_sec++;
        w.deadline.tv_nsec -= 1000000000;
    }

    pthread_mutex_lock(&res->mutex);

    // 대기 큐에 추가 (FIFO 보장)
    if (res->tail) {
        res->tail->next = &w;
    } else {
        res->head = &w;
    }
    res->tail = &w;

    // 내 차례까지 대기
    while (!w.ready) {
        int rc = pthread_cond_timedwait(&res->cond, &res->mutex,
                                        &w.deadline);
        if (rc == ETIMEDOUT) {
            // 큐에서 제거
            remove_waiter(res, &w);
            pthread_mutex_unlock(&res->mutex);
            return -ETIMEDOUT;
        }
    }

    res->in_use = true;
    pthread_mutex_unlock(&res->mutex);
    return 0;
}
```

## Detection Strategies

### 1. Wait Time Monitoring

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

### 2. Fairness Metrics

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

### 3. Queue Length Tracking

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

## Fairness Concepts

### Strong Fairness

Every thread that wants access will eventually get it.

```c
// Example: FIFO mutex (shown earlier)
// Guarantees: If thread requests lock, it WILL get it
```

### Weak Fairness

If a thread keeps wanting access, it will eventually get it.

```c
// Example: Simple mutex with no guarantees
// Only ensures: continuous requests eventually succeed
```

### No Fairness

No guarantees about who gets access when.

```c
// Example: Spinlock without queue
while (!atomic_compare_exchange(&lock, &expected, 1)) {
    // Any thread might win - no fairness
}
```

### Fairness Comparison

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

## Best Practices

### DO:
- ✓ Use fair synchronization primitives
- ✓ Monitor wait times and detect starvation
- ✓ Implement aging for priority systems
- ✓ Bound priority ranges
- ✓ Use FIFO queues where possible
- ✓ Set timeout limits
- ✓ Test under high load conditions

### DON'T:
- ✗ Use unbounded priorities
- ✗ Always prefer one class of threads
- ✗ Ignore wait time metrics
- ✗ Assume fairness without verification
- ✗ Use pure priority scheduling for long-running tasks

## Summary

**Starvation** occurs when threads are perpetually denied resources due to:
- Unfair scheduling
- Priority schemes
- Reader-writer imbalance
- Lack of fairness guarantees

**Key Differences:**
```
Deadlock:    No progress by anyone
Livelock:    Activity but no progress
Starvation:  Some progress, but not by everyone
```

**Prevention Strategies:**
1. Fair locks (FIFO ordering)
2. Aging algorithms
3. Bounded waiting
4. Round-robin scheduling
5. Fairness monitoring

## Exercises

### Exercise 1: Detect Starvation
Add monitoring to detect when a thread has waited more than 5 seconds.

### Exercise 2: Implement Fair Queue
Create a fair priority queue where old low-priority items eventually get served.

### Exercise 3: Fix Reader Starvation
Modify the reader-writer lock to prevent writer starvation.

## Further Reading

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Modern Operating Systems" - Andrew Tanenbaum

## Next Topic

Continue to [05-priority-inversion.md](./05-priority-inversion.md) to learn about priority inversion.
