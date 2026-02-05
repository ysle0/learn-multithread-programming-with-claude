# Starvation

## Starvation이란?

**Starvation**은 thread가 진행에 필요한 리소스에 대한 접근을 영구적으로 거부당할 때 발생합니다. 모든 thread가 멈추는 deadlock과 달리, starvation에서는 일부 thread는 진행되는 반면 다른 thread는 무기한으로 지연됩니다. Starvation에 빠진 thread는 결국 리소스를 얻을 수도 있지만, 대기 시간은 제한이 없고 예측할 수 없습니다.

### 정의

thread가 starvation을 겪는 조건:
1. 실행 준비가 되어 있고 리소스가 필요한 상태
2. 다른 thread들이 지속적으로 해당 리소스를 획득
3. thread가 진행 없이 무기한 대기
4. 시스템 전체는 진행됨 (deadlock과 다른 점)

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

### Starvation과 다른 문제의 비교

```
┌───────────────┬──────────┬──────────┬────────────┬──────────┐
│   Problem     │  Blocked │ Progress │   Cause    │ Severity │
├───────────────┼──────────┼──────────┼────────────┼──────────┤
│ Deadlock      │   All    │   None   │  Circular  │ Critical │
│ Livelock      │   None   │   None   │  Collision │   High   │
│ Starvation    │   Some   │  Partial │  Unfair    │  Medium  │
└───────────────┴──────────┴──────────┴────────────┴──────────┘
```

## Starvation의 일반적인 원인

### 1. 우선순위 기반 스케줄링

높은 우선순위의 thread가 항상 낮은 우선순위의 thread를 선점합니다.

```c
#include <pthread.h>
#include <stdio.h>
#include <sched.h>

pthread_mutex_t resource = PTHREAD_MUTEX_INITIALIZER;

void* high_priority_thread(void* arg) {
    // 높은 우선순위 설정
    struct sched_param param;
    param.sched_priority = 99;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("High priority: Working\n");
        // 작업 수행...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

void* low_priority_thread(void* arg) {
    // 낮은 우선순위 설정
    struct sched_param param;
    param.sched_priority = 1;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("Low priority: Working\n");  // 출력되지 않을 수 있음
        // 작업 수행...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

// 높은 우선순위 thread가 지속적으로 실행되면 낮은 우선순위 thread가 STARVATION에 빠질 수 있음
```

### 2. 불공정한 Lock 구현

일부 lock 구현은 공정성을 보장하지 않습니다.

```c
// 불공정한 mutex 구현 (간략화)
typedef struct {
    atomic_int locked;
    // 큐 없음 - thread들이 경쟁적으로 획득 시도
} UnfairMutex;

void unfair_lock(UnfairMutex* m) {
    // 성공할 때까지 spin
    while (1) {
        int expected = 0;
        if (atomic_compare_exchange_weak(&m->locked, &expected, 1)) {
            return;  // 획득 성공
        }
        // 일부 thread가 다른 thread보다 더 빨리 재시도할 수 있음!
        // 빠른 thread가 느린 thread를 starvation에 빠뜨릴 수 있음
    }
}

// 더 빠른 CPU 코어를 가진 thread가 항상 이길 수 있음
// 느린 코어를 가진 thread는 STARVATION에 빠질 수 있음
```

### 3. Reader-Writer 문제

Reader가 계속 도착하면 writer가 starvation에 빠질 수 있습니다.

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

    // 모든 reader가 완료될 때까지 대기
    while (lock->readers > 0) {
        pthread_mutex_unlock(&lock->mutex);
        sched_yield();
        pthread_mutex_lock(&lock->mutex);
    }

    // 이제 쓰기 접근 권한 획득
}

// 문제: Reader가 계속 도착하면 writer가 STARVATION에 빠짐
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

### 4. 불공정한 Semaphore를 사용한 Producer-Consumer

```c
#include <semaphore.h>
#include <pthread.h>

#define BUFFER_SIZE 10

sem_t empty;  // 빈 슬롯 수
sem_t full;   // 찬 슬롯 수

void* producer(void* arg) {
    while (1) {
        sem_wait(&empty);  // 빈 슬롯 대기

        // 아이템 생산
        produce_item();

        sem_post(&full);   // 아이템 사용 가능 신호
    }
    return NULL;
}

void* consumer(void* arg) {
    while (1) {
        sem_wait(&full);   // 아이템 대기

        // 아이템 소비
        consume_item();

        sem_post(&empty);  // 슬롯 비어있음 신호
    }
    return NULL;
}

// 빠른 producer가 많고 느린 consumer가 하나이면,
// 느린 consumer가 STARVATION에 빠질 수 있음
```

## 카테고리별 예제

### 예제 1: Thread Pool Starvation

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

#define NUM_WORKERS 4
#define QUEUE_SIZE 100

typedef struct {
    void (*function)(void*);
    void* arg;
    int priority;  // 높을수록 더 중요
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

    // 우선순위별 삽입 (높은 우선순위 먼저)
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

        // 가장 높은 우선순위 작업 가져오기
        Task task = pool.queue[0];
        pool.size--;

        // 큐 이동
        for (int i = 0; i < pool.size; i++) {
            pool.queue[i] = pool.queue[i+1];
        }

        pthread_mutex_unlock(&pool.mutex);

        // 작업 실행
        task.function(task.arg);
    }
    return NULL;
}

// 문제: 높은 우선순위 작업이 계속 도착하면
// 낮은 우선순위 작업이 STARVATION에 빠질 수 있음
```

**시각화:**
```
Queue State (priority-ordered):
[9][9][9][8][8][7][7][7][6][5] ← High priority kept arriving
                               [2] ← Low priority task STARVING
```

### 예제 2: 디스크 I/O Scheduler Starvation

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int track;      // 디스크 트랙 번호
    int timestamp;  // 요청 도착 시각
} IORequest;

// SCAN (엘리베이터) 알고리즘 - starvation을 유발할 수 있음
void scan_schedule(IORequest* requests, int count, int current_track) {
    int direction = 1;  // 1 = 위, -1 = 아래

    while (1) {
        int served = 0;

        // 현재 방향의 요청 처리
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
            direction = -direction;  // 방향 반전
        }

        // 문제: 현재 쪽에 새 요청이 계속 도착하면
        // 반대쪽 끝의 요청이 STARVATION에 빠질 수 있음
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

        // 항상 가장 높은 우선순위 먼저 처리
        int highest_idx = 0;
        for (int i = 1; i < queue_size; i++) {
            if (queue[i].priority > queue[highest_idx].priority) {
                highest_idx = i;
            }
        }

        Packet p = queue[highest_idx];

        // Starvation 확인
        uint64_t wait_time = current_time() - p.arrival_time;
        if (wait_time > 10000) {  // 10초
            printf("WARNING: Packet starved for %llu ms\n", wait_time);
        }

        process_packet(&p);

        // 큐에서 제거
        queue[highest_idx] = queue[--queue_size];
    }
}

// 높은 우선순위 패킷이 계속 도착하면 낮은 우선순위 패킷이 STARVATION에 빠짐
```

## 해결 방법 및 예방

### 해결 방법 1: 공정한 Mutex (FIFO 순서)

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
        return;  // 즉시 lock 획득
    }

    // 대기 큐에 추가
    if (fm->tail) {
        fm->tail->next = &node;
    } else {
        fm->head = &node;
    }
    fm->tail = &node;

    // 차례 대기
    while (!node.ready) {
        pthread_cond_wait(&node.cond, &fm->mutex);
    }

    pthread_mutex_unlock(&fm->mutex);
    pthread_cond_destroy(&node.cond);
}

void fair_mutex_unlock(FairMutex* fm) {
    pthread_mutex_lock(&fm->mutex);

    if (fm->head) {
        // 다음 대기자 깨우기
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

// FIFO 순서가 starvation을 방지함
```

### 해결 방법 2: 공정한 Reader-Writer Lock

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

    // Writer가 있거나 대기 중인 writer가 있으면 대기
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
        // 대기 중인 writer 깨우기
        pthread_cond_signal(&lock->writers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

void fair_write_lock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->waiting_writers++;

    // Reader와 writer가 끝날 때까지 대기
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
        // 대기 중인 writer 우선
        pthread_cond_signal(&lock->writers_cond);
    } else {
        // 모든 대기 중인 reader 깨우기
        pthread_cond_broadcast(&lock->readers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

// Writer가 starvation에 빠지지 않음 - 현재 reader 이후에 우선 처리됨
```

### 해결 방법 3: Aging 우선순위

대기 중인 thread의 우선순위를 시간이 지남에 따라 증가시킵니다.

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
    // 10초마다 우선순위 1 증가
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

// 오래된 낮은 우선순위 작업이 결국 높은 우선순위가 됨
// 무기한 starvation 방지
```

### 해결 방법 4: Round-Robin 스케줄링

각 thread에 시간 할당량을 부여합니다.

```c
#include <pthread.h>
#include <signal.h>
#include <time.h>

#define NUM_THREADS 5
#define TIME_SLICE_MS 100

pthread_t threads[NUM_THREADS];
int current_thread = 0;

void switch_thread(int sig) {
    // 현재 thread 일시정지
    pthread_kill(threads[current_thread], SIGSTOP);

    // 다음 thread로 전환
    current_thread = (current_thread + 1) % NUM_THREADS;

    // 다음 thread 재개
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

// 모든 thread가 동일한 CPU 시간을 받음 - starvation 없음
```

### 해결 방법 5: 2단계 피드백 큐

```c
#define NUM_QUEUES 3

typedef struct {
    Task queues[NUM_QUEUES][100];
    int sizes[NUM_QUEUES];
    int execution_counts[1000];  // thread별 추적
} FeedbackQueue;

FeedbackQueue fbq = {{{0}}, {0}, {0}};

void enqueue_with_feedback(int thread_id, Task task) {
    // 새 작업은 가장 높은 우선순위 큐에서 시작
    int queue_level = 0;

    // 너무 많이 실행되면 강등
    int exec_count = fbq.execution_counts[thread_id];
    if (exec_count > 10) queue_level = 2;      // 낮은 우선순위
    else if (exec_count > 3) queue_level = 1;  // 중간 우선순위

    fbq.queues[queue_level][fbq.sizes[queue_level]++] = task;
}

Task* get_next_with_feedback() {
    // 높은 우선순위 큐부터 처리
    for (int level = 0; level < NUM_QUEUES; level++) {
        if (fbq.sizes[level] > 0) {
            Task* task = &fbq.queues[level][0];

            // 큐에서 제거
            for (int i = 0; i < fbq.sizes[level] - 1; i++) {
                fbq.queues[level][i] = fbq.queues[level][i + 1];
            }
            fbq.sizes[level]--;

            return task;
        }
    }
    return NULL;
}

// 높은 우선순위 큐가 비어있으면 낮은 우선순위 작업도 처리됨
```

## 내부 메커니즘

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

### 2. 공정성 지표

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

### 강한 공정성 (Strong Fairness)

접근을 원하는 모든 thread는 결국 접근 권한을 얻습니다.

```c
// 예시: FIFO mutex (앞에서 설명)
// 보장: thread가 lock을 요청하면, 반드시 획득함
```

### 약한 공정성 (Weak Fairness)

thread가 계속 접근을 원하면, 결국 접근 권한을 얻습니다.

```c
// 예시: 보장이 없는 단순 mutex
// 보장: 지속적인 요청은 결국 성공함
```

### 공정성 없음 (No Fairness)

누가 언제 접근 권한을 얻는지에 대한 보장이 없습니다.

```c
// 예시: 큐 없는 spinlock
while (!atomic_compare_exchange(&lock, &expected, 1)) {
    // 어떤 thread든 이길 수 있음 - 공정성 없음
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

### 권장 사항:
- ✓ 공정한 동기화 프리미티브 사용
- ✓ 대기 시간 모니터링 및 starvation 탐지
- ✓ 우선순위 시스템에 aging 구현
- ✓ 우선순위 범위 제한
- ✓ 가능한 곳에 FIFO 큐 사용
- ✓ 타임아웃 제한 설정
- ✓ 높은 부하 조건에서 테스트

### 금지 사항:
- ✗ 제한 없는 우선순위 사용
- ✗ 한 종류의 thread를 항상 우선시
- ✗ 대기 시간 지표 무시
- ✗ 검증 없이 공정성 가정
- ✗ 장시간 실행되는 작업에 순수 우선순위 스케줄링 사용

## 요약

**Starvation**은 다음과 같은 이유로 thread가 영구적으로 리소스를 거부당할 때 발생합니다:
- 불공정한 스케줄링
- 우선순위 체계
- Reader-writer 불균형
- 공정성 보장 부재

**주요 차이점:**
```
Deadlock:    아무도 진행하지 못함
Livelock:    활동은 있지만 진행 없음
Starvation:  일부는 진행되지만 모두가 진행되지는 않음
```

**예방 전략:**
1. 공정한 lock (FIFO 순서)
2. Aging 알고리즘
3. 제한된 대기
4. Round-robin 스케줄링
5. 공정성 모니터링

## 연습 문제

### 연습 문제 1: Starvation 탐지
thread가 5초 이상 대기했을 때 이를 탐지하는 모니터링을 추가하세요.

### 연습 문제 2: 공정한 큐 구현
오래된 낮은 우선순위 항목이 결국 처리되는 공정한 우선순위 큐를 만드세요.

### 연습 문제 3: Reader Starvation 수정
Writer starvation을 방지하도록 reader-writer lock을 수정하세요.

## 추가 참고 자료

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Modern Operating Systems" - Andrew Tanenbaum

## 다음 주제

[05-priority-inversion.md](./05-priority-inversion.md)에서 priority inversion에 대해 알아보세요.
