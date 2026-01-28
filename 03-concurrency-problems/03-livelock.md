# Livelock

## What is Livelock?

**Livelock** is a situation where threads are not blocked (unlike deadlock), but they continuously change state in response to each other without making any meaningful progress. Threads remain active and consume CPU resources, but the system as a whole doesn't advance toward completion.

Think of it as two people trying to pass each other in a narrow hallway - they both step to the same side simultaneously, then both step to the other side, repeating forever without actually passing.

## Livelock vs Deadlock

```
┌──────────────────┬───────────────────┬──────────────────┐
│   Characteristic │     Deadlock      │     Livelock     │
├──────────────────┼───────────────────┼──────────────────┤
│  Thread State    │     Blocked       │      Active      │
│  CPU Usage       │       None        │       High       │
│  Progress        │       None        │       None       │
│  Detection       │     Easier        │      Harder      │
│  Visibility      │   Threads stuck   │  Threads busy    │
│  Resource Usage  │   Held/Locked     │   Released/Retry │
└──────────────────┴───────────────────┴──────────────────┘
```

### Visual Comparison

**Deadlock:**
```
Thread 1: [BLOCKED] ━━━━━━━━━━━━━━━━━ (waiting forever)
Thread 2: [BLOCKED] ━━━━━━━━━━━━━━━━━ (waiting forever)

CPU: Idle
Progress: NONE
```

**Livelock:**
```
Thread 1: [ACTIVE] ──↺──↺──↺──↺──↺── (busy but no progress)
Thread 2: [ACTIVE] ──↺──↺──↺──↺──↺── (busy but no progress)

CPU: 100% busy
Progress: NONE
```

## Classic Example: The Hallway Problem

```
Person A ←─────────────────→ Person B
         Narrow Hallway

Step 1: A moves left, B moves left   (both still blocked)
Step 2: A moves right, B moves right (both still blocked)
Step 3: A moves left, B moves left   (both still blocked)
...repeats forever...
```

### Code Implementation

```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>
#include <unistd.h>

typedef struct {
    bool trying_left;
    bool trying_right;
    int id;
} Person;

Person person_a = {false, false, 1};
Person person_b = {false, false, 2};

void* person_a_walk(void* arg) {
    while (true) {
        if (person_b.trying_left) {
            printf("Person A: B is on left, I'll go left too\n");
            person_a.trying_left = true;
            usleep(100000);  // 100ms
        } else if (person_b.trying_right) {
            printf("Person A: B is on right, I'll go right too\n");
            person_a.trying_right = true;
            usleep(100000);
        }

        // Reset and try again
        person_a.trying_left = false;
        person_a.trying_right = false;
    }
    return NULL;
}

void* person_b_walk(void* arg) {
    while (true) {
        if (person_a.trying_left) {
            printf("Person B: A is on left, I'll go left too\n");
            person_b.trying_left = true;
            usleep(100000);
        } else if (person_a.trying_right) {
            printf("Person B: A is on right, I'll go right too\n");
            person_b.trying_right = true;
            usleep(100000);
        }

        // Reset and try again
        person_b.trying_left = false;
        person_b.trying_right = false;
    }
    return NULL;
}

// This creates LIVELOCK - both keep moving but never pass!
```

## Common Livelock Patterns

### Pattern 1: Collision Avoidance

When threads detect conflicts and back off, but do so in a synchronized manner.

```c
#include <pthread.h>
#include <stdbool.h>
#include <stdio.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_livelock(void* arg) {
    int id = *(int*)arg;

    while (true) {
        // Try to acquire both resources
        pthread_mutex_lock(&resource_a);

        if (pthread_mutex_trylock(&resource_b) != 0) {
            // Failed to get B, release A and retry
            printf("Thread %d: Failed to get B, releasing A\n", id);
            pthread_mutex_unlock(&resource_a);

            // PROBLEM: Both threads do this simultaneously!
            // They keep releasing and retrying forever
            continue;
        }

        // Critical section
        printf("Thread %d: Got both resources!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}

// LIVELOCK: If both threads retry at same time, they collide repeatedly
```

**Timeline:**
```
Time    Thread 1                Thread 2
----    --------                --------
  1     Lock A                  Lock A (wait)
  2     Try B (fail)            -
  3     Unlock A                Lock A (acquired)
  4     -                       Try B (fail)
  5     Lock A (wait)           Unlock A
  6     Lock A (acquired)       Lock A (wait)
  7     Try B (fail)            -
  8     ...repeats...           ...repeats...
```

### Pattern 2: Polite Threads

Threads try to be "polite" and yield to others, but all do it simultaneously.

```c
#include <pthread.h>
#include <stdbool.h>
#include <sched.h>

volatile bool thread1_wants = false;
volatile bool thread2_wants = false;

void* polite_thread1(void* arg) {
    while (true) {
        thread1_wants = true;

        // Be polite: if other thread wants it, yield
        while (thread2_wants) {
            thread1_wants = false;  // Give way
            sched_yield();          // Let other thread go
            thread1_wants = true;   // Want it again
        }

        // Critical section
        critical_section();

        thread1_wants = false;
    }
    return NULL;
}

void* polite_thread2(void* arg) {
    while (true) {
        thread2_wants = true;

        // Be polite: if other thread wants it, yield
        while (thread1_wants) {
            thread2_wants = false;  // Give way
            sched_yield();          // Let other thread go
            thread2_wants = true;   // Want it again
        }

        // Critical section
        critical_section();

        thread2_wants = false;
    }
    return NULL;
}

// LIVELOCK: Both keep yielding to each other!
```

### Pattern 3: Message Retransmission

In distributed systems, nodes retransmit on collision but create more collisions.

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <stdbool.h>

typedef struct {
    int id;
    int attempts;
} Node;

bool try_send(Node* node) {
    // Simulate collision detection
    bool collision = (rand() % 2 == 0);

    if (collision) {
        printf("Node %d: Collision detected, retry attempt %d\n",
               node->id, node->attempts);
        node->attempts++;
        return false;
    }

    printf("Node %d: Sent successfully!\n", node->id);
    return true;
}

void node_send_with_livelock(Node* node) {
    while (!try_send(node)) {
        // Fixed retry interval - causes synchronized retries
        usleep(1000);  // Always wait 1ms

        // LIVELOCK: All nodes retry at same time!
    }
}
```

## Solutions and Prevention

### Solution 1: Random Backoff

Introduce randomness to break synchronization.

```c
#include <pthread.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_random_backoff(void* arg) {
    int id = *(int*)arg;
    srand(time(NULL) + id);  // Different seed per thread

    while (true) {
        pthread_mutex_lock(&resource_a);

        if (pthread_mutex_trylock(&resource_b) != 0) {
            pthread_mutex_unlock(&resource_a);

            // Random backoff: 0-10ms
            int backoff = rand() % 10000;
            printf("Thread %d: Backing off %dμs\n", id, backoff);
            usleep(backoff);
            continue;
        }

        // Critical section
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

### Solution 2: Exponential Backoff

Increase backoff time with each retry (like Ethernet CSMA/CD).

```c
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>

#define MAX_BACKOFF 1000000  // 1 second

void* thread_with_exponential_backoff(void* arg) {
    int id = *(int*)arg;
    int backoff = 1000;  // Start with 1ms

    while (true) {
        pthread_mutex_lock(&resource_a);

        if (pthread_mutex_trylock(&resource_b) != 0) {
            pthread_mutex_unlock(&resource_a);

            printf("Thread %d: Backing off %dμs\n", id, backoff);
            usleep(backoff);

            // Exponential backoff
            backoff = (backoff * 2 < MAX_BACKOFF) ? backoff * 2 : MAX_BACKOFF;
            continue;
        }

        // Critical section
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

### Solution 3: Priority-Based Resolution

Give one thread higher priority.

```c
#include <pthread.h>
#include <stdbool.h>

typedef struct {
    int id;
    int priority;
} ThreadInfo;

volatile bool low_priority_wants = false;
volatile bool high_priority_wants = false;

void* low_priority_thread(void* arg) {
    while (true) {
        low_priority_wants = true;

        // Yield to high priority thread
        while (high_priority_wants) {
            low_priority_wants = false;
            sched_yield();
            low_priority_wants = true;
        }

        // Critical section
        critical_section();
        low_priority_wants = false;
    }
    return NULL;
}

void* high_priority_thread(void* arg) {
    while (true) {
        high_priority_wants = true;

        // Don't yield - take priority!
        // Critical section
        critical_section();
        high_priority_wants = false;
    }
    return NULL;
}

// NO LIVELOCK: High priority always proceeds
```

### Solution 4: Lock Ordering

Use consistent lock ordering to avoid retries.

```c
#include <pthread.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_ordering(void* arg) {
    // Always acquire in order: A then B
    pthread_mutex_lock(&resource_a);
    pthread_mutex_lock(&resource_b);

    // Critical section
    critical_section();

    pthread_mutex_unlock(&resource_b);
    pthread_mutex_unlock(&resource_a);

    return NULL;
}

// NO LIVELOCK: No trylock, no retries needed
```

### Solution 5: Timeout with Randomization

Combine timeout with random retry.

```c
#include <pthread.h>
#include <time.h>
#include <errno.h>
#include <stdlib.h>

void* thread_with_timeout(void* arg) {
    int id = *(int*)arg;

    while (true) {
        struct timespec timeout;
        clock_gettime(CLOCK_REALTIME, &timeout);
        timeout.tv_sec += 1;  // 1 second timeout

        pthread_mutex_lock(&resource_a);

        int result = pthread_mutex_timedlock(&resource_b, &timeout);

        if (result == ETIMEDOUT) {
            pthread_mutex_unlock(&resource_a);

            // Random backoff before retry
            usleep(rand() % 100000);
            continue;
        }

        // Critical section
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

## Real-World Examples

### Example 1: Network Collision (Ethernet)

```c
// Simplified CSMA/CD (Carrier Sense Multiple Access with Collision Detection)
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

#define MAX_ATTEMPTS 16

typedef struct {
    int id;
    int collisions;
} NetworkNode;

void transmit_with_csma_cd(NetworkNode* node) {
    int attempt = 0;

    while (attempt < MAX_ATTEMPTS) {
        // Listen for carrier
        if (channel_busy()) {
            wait_until_idle();
        }

        // Transmit
        if (send_frame()) {
            printf("Node %d: Transmission successful\n", node->id);
            return;
        }

        // Collision detected
        node->collisions++;
        printf("Node %d: Collision #%d\n", node->id, node->collisions);

        // Binary exponential backoff
        int k = (attempt < 10) ? attempt : 10;
        int backoff_slots = rand() % (1 << k);  // 0 to 2^k - 1
        usleep(backoff_slots * 512);  // 512μs per slot

        attempt++;
    }

    printf("Node %d: Failed after %d attempts\n", node->id, MAX_ATTEMPTS);
}
```

### Example 2: Database Retry Logic

```c
#include <pthread.h>
#include <stdbool.h>
#include <stdlib.h>
#include <time.h>

typedef struct {
    pthread_mutex_t mutex;
    int value;
} DBRecord;

DBRecord records[1000];

bool update_records_with_livelock(int id1, int id2, int delta) {
    int attempts = 0;

    while (attempts < 100) {
        pthread_mutex_lock(&records[id1].mutex);

        if (pthread_mutex_trylock(&records[id2].mutex) != 0) {
            // Deadlock avoidance causes livelock!
            pthread_mutex_unlock(&records[id1].mutex);
            attempts++;
            continue;  // Fixed retry = LIVELOCK
        }

        // Update both records
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;  // Failed
}

bool update_records_fixed(int id1, int id2, int delta) {
    int attempts = 0;

    while (attempts < 100) {
        pthread_mutex_lock(&records[id1].mutex);

        if (pthread_mutex_trylock(&records[id2].mutex) != 0) {
            pthread_mutex_unlock(&records[id1].mutex);

            // Random backoff prevents livelock
            usleep(rand() % 10000);
            attempts++;
            continue;
        }

        // Update both records
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;
}
```

### Example 3: Distributed Consensus

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

typedef struct {
    int id;
    int proposed_value;
    int seen_proposals;
} Node;

// Simplified consensus with livelock potential
void reach_consensus_bad(Node* nodes, int num_nodes) {
    bool consensus_reached = false;

    while (!consensus_reached) {
        // Each node proposes its value
        for (int i = 0; i < num_nodes; i++) {
            nodes[i].seen_proposals = 0;

            // Check what others proposed
            for (int j = 0; j < num_nodes; j++) {
                if (nodes[j].proposed_value == nodes[i].proposed_value) {
                    nodes[i].seen_proposals++;
                }
            }

            // If not majority, change proposal
            if (nodes[i].seen_proposals < num_nodes / 2) {
                // Pick random new value
                nodes[i].proposed_value = rand() % 100;
                printf("Node %d: Changing proposal\n", i);
            }
        }

        // Check for consensus
        int first_value = nodes[0].proposed_value;
        consensus_reached = true;
        for (int i = 1; i < num_nodes; i++) {
            if (nodes[i].proposed_value != first_value) {
                consensus_reached = false;
                break;
            }
        }
    }

    // LIVELOCK: Nodes keep changing proposals!
}

// Fixed version with leader election
void reach_consensus_good(Node* nodes, int num_nodes) {
    // Elect leader (e.g., lowest ID)
    int leader_id = 0;
    for (int i = 1; i < num_nodes; i++) {
        if (nodes[i].id < nodes[leader_id].id) {
            leader_id = i;
        }
    }

    // Everyone adopts leader's proposal
    int consensus_value = nodes[leader_id].proposed_value;
    for (int i = 0; i < num_nodes; i++) {
        nodes[i].proposed_value = consensus_value;
    }

    printf("Consensus reached: %d\n", consensus_value);
    // NO LIVELOCK: Single decision maker
}
```

## Internal Mechanisms

### 스케줄러 관점에서의 Livelock

Linux CFS(Completely Fair Scheduler)가 livelock 상황을 어떻게 처리하는지 살펴봅니다.

#### sched_yield() 내부 동작

```c
// linux/kernel/sched/core.c
SYSCALL_DEFINE0(sched_yield)
{
    struct rq *rq = this_rq();

    // 런큐 락 획득
    raw_spin_lock_irq(&rq->lock);

    // 현재 태스크를 런큐 끝으로 이동
    schedstat_inc(rq->yld_count);

    // CFS: vruntime을 최소값으로 설정하지 않음
    // 대신 현재 vruntime 유지 → 공정성 보장
    current->se.vruntime += sched_min_granularity;

    // 재스케줄링 요청
    set_need_resched();

    raw_spin_unlock_irq(&rq->lock);

    // 즉시 스케줄러 호출
    schedule();

    return 0;
}
```

#### Livelock 발생 시 CPU 상태

```
┌─────────────────────────────────────────────────────────────┐
│                    Livelock CPU State                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CPU 0                          CPU 1                       │
│  ┌─────────────────────┐       ┌─────────────────────┐     │
│  │ Thread A            │       │ Thread B            │     │
│  │ State: RUNNING      │       │ State: RUNNING      │     │
│  │ CPU: 100%           │       │ CPU: 100%           │     │
│  │                     │       │                     │     │
│  │ trylock(mutex_b)    │       │ trylock(mutex_a)    │     │
│  │ → EBUSY             │       │ → EBUSY             │     │
│  │ unlock(mutex_a)     │       │ unlock(mutex_b)     │     │
│  │ yield()             │       │ yield()             │     │
│  │ lock(mutex_a)       │       │ lock(mutex_b)       │     │
│  │ ... 반복 ...        │       │ ... 반복 ...        │     │
│  └─────────────────────┘       └─────────────────────┘     │
│                                                             │
│  특징: TASK_RUNNING 상태이지만 유용한 작업 없음            │
│        context switch 빈번 발생 (캐시 효율 저하)           │
└─────────────────────────────────────────────────────────────┘
```

### 커널 레벨 Livelock 탐지

```c
// 커널에서 사용하는 livelock 탐지 기법 (예: 네트워크 스택)
// linux/net/core/dev.c

static void check_net_livelock(struct net_device *dev) {
    // NAPI polling에서의 livelock 방지

    // 작업량 제한 (budget)
    int budget = netdev_budget;  // 기본값: 300

    // 한 번의 poll에서 최대 budget 만큼만 처리
    int work_done = dev->poll(dev, budget);

    if (work_done >= budget) {
        // 아직 처리할 패킷이 더 있음
        // 다른 장치에게도 기회 제공 (공정성)
        schedule_delayed_work(&dev->poll_work, 1);

        // livelock 카운터 증가
        dev->livelock_count++;

        if (dev->livelock_count > LIVELOCK_THRESHOLD) {
            // 경고 출력
            netdev_warn(dev, "Possible livelock detected, "
                       "throttling packet processing\n");

            // 처리 속도 조절
            dev->poll_budget = budget / 2;
        }
    } else {
        // 모든 패킷 처리 완료
        dev->livelock_count = 0;
    }
}
```

### Exponential Backoff 구현 세부사항

```c
// 이더넷 CSMA/CD 스타일 backoff
// 실제 구현에서의 고려사항

struct backoff_state {
    int attempt;          // 현재 시도 횟수
    int max_attempts;     // 최대 시도 (보통 16)
    int slot_time_us;     // 슬롯 시간 (이더넷: 51.2μs)
    uint32_t seed;        // 스레드별 난수 시드
};

int exponential_backoff(struct backoff_state *state) {
    if (state->attempt >= state->max_attempts) {
        return -ETIMEDOUT;  // 포기
    }

    // k = min(attempt, 10)
    int k = (state->attempt < 10) ? state->attempt : 10;

    // 0 ~ (2^k - 1) 범위의 난수 선택
    int max_slots = (1 << k) - 1;
    int slots = fast_random(&state->seed) % (max_slots + 1);

    // 대기 시간 계산
    int wait_time_us = slots * state->slot_time_us;

    // 실제 대기 (busy-wait 대신 sleep 사용)
    if (wait_time_us > 0) {
        struct timespec ts = {
            .tv_sec = wait_time_us / 1000000,
            .tv_nsec = (wait_time_us % 1000000) * 1000
        };
        nanosleep(&ts, NULL);
    }

    state->attempt++;
    return 0;
}

// 빠른 난수 생성 (xorshift)
static inline uint32_t fast_random(uint32_t *seed) {
    uint32_t x = *seed;
    x ^= x << 13;
    x ^= x >> 17;
    x ^= x << 5;
    *seed = x;
    return x;
}
```

### Lock-Free 환경에서의 Livelock

```c
// CAS 루프에서의 livelock 가능성

// 문제: 여러 스레드가 동시에 CAS 시도
void problematic_increment(atomic_int *counter) {
    int old_val, new_val;

    do {
        old_val = atomic_load(counter);
        new_val = old_val + 1;
        // 모든 스레드가 동시에 실패 → livelock 유사 상황
    } while (!atomic_compare_exchange_weak(counter, &old_val, new_val));
}

// 해결: Backoff 적용
void increment_with_backoff(atomic_int *counter) {
    struct backoff_state bs = {
        .attempt = 0,
        .max_attempts = 16,
        .slot_time_us = 1,
        .seed = (uint32_t)pthread_self()
    };

    int old_val, new_val;

    do {
        old_val = atomic_load(counter);
        new_val = old_val + 1;

        if (atomic_compare_exchange_weak(counter, &old_val, new_val)) {
            return;  // 성공
        }

        // CAS 실패 시 backoff
        if (exponential_backoff(&bs) < 0) {
            // 포기 - 다른 전략 사용 (예: 락 기반)
            fallback_to_lock(counter);
            return;
        }
    } while (1);
}
```

### 스핀락에서의 Livelock 방지

```c
// Linux 커널의 ticket spinlock (공정성 보장)
// arch/x86/include/asm/spinlock.h (개념적)

typedef struct {
    atomic_int head;  // 서비스 중인 번호
    atomic_int tail;  // 다음 발급 번호
} ticket_spinlock_t;

void ticket_spin_lock(ticket_spinlock_t *lock) {
    // 번호표 받기 (원자적)
    int my_ticket = atomic_fetch_add(&lock->tail, 1);

    // 내 차례 대기
    while (atomic_load(&lock->head) != my_ticket) {
        // PAUSE 명령으로 CPU 절전 + 파이프라인 최적화
        cpu_relax();  // x86: PAUSE instruction

        // 선택적: 긴 대기 시 yield
        if (should_yield()) {
            sched_yield();
        }
    }

    // 메모리 배리어
    smp_mb();
}

void ticket_spin_unlock(ticket_spinlock_t *lock) {
    smp_mb();
    // 다음 번호 호출
    atomic_fetch_add(&lock->head, 1);
}

// 장점:
// - FIFO 순서 보장 → livelock/starvation 방지
// - 공정한 대기 시간
```

### perf를 이용한 Livelock 분석

```bash
# Livelock 의심 상황에서의 분석

# 1. context switch 횟수 확인
perf stat -e context-switches,cpu-migrations \
    -p <pid> sleep 5

# 출력 예시 (livelock 시):
#  1,234,567 context-switches    # 매우 높음!
#      1,234 cpu-migrations

# 2. 함수별 CPU 시간 분석
perf record -g -p <pid> sleep 10
perf report

# livelock 시 특정 함수에서 대부분의 시간 소모:
# 45% pthread_mutex_trylock
# 40% pthread_mutex_unlock
# 10% sched_yield
#  5% 실제 작업

# 3. 락 경합 분석
perf lock record -p <pid>
perf lock report

# 4. 실시간 모니터링
watch -n 1 "cat /proc/<pid>/status | grep -E '(State|voluntary|nonvoluntary)'"

# livelock 시:
# State: R (running)
# voluntary_ctxt_switches: 매우 높음
# nonvoluntary_ctxt_switches: 낮음
```

## Detection Strategies

### 1. Progress Monitoring

```c
#include <time.h>

typedef struct {
    int work_completed;
    time_t last_progress;
} ProgressMonitor;

ProgressMonitor monitor = {0, 0};

void check_for_livelock() {
    time_t now = time(NULL);

    if (monitor.work_completed == last_work &&
        now - monitor.last_progress > 5) {
        printf("LIVELOCK suspected: No progress in 5 seconds\n");
        printf("Threads active but not advancing\n");
    }

    monitor.last_progress = now;
}
```

### 2. Retry Counter

```c
#define MAX_RETRIES 1000

int retry_count = 0;

void detect_excessive_retries() {
    retry_count++;

    if (retry_count > MAX_RETRIES) {
        printf("LIVELOCK suspected: %d retries!\n", retry_count);
        // Take corrective action
        abort();
    }
}
```

### 3. CPU Usage Analysis

```bash
# Monitor CPU usage
top -H -p <pid>

# If threads show high CPU but no progress → livelock

# Use perf to see what threads are doing
perf record -p <pid> -g
perf report
```

## Prevention Best Practices

### Checklist

- [ ] Use random backoff instead of fixed delays
- [ ] Implement exponential backoff for retries
- [ ] Set maximum retry limits
- [ ] Use lock ordering instead of trylock when possible
- [ ] Add timeout mechanisms
- [ ] Monitor progress metrics
- [ ] Test with multiple threads under load
- [ ] Avoid symmetric retry logic

### Design Patterns

**Pattern 1: Asymmetric Behavior**
```c
void* thread_function(void* arg) {
    int id = *(int*)arg;

    // Even threads use one strategy
    if (id % 2 == 0) {
        strategy_a();
    }
    // Odd threads use another
    else {
        strategy_b();
    }
}
```

**Pattern 2: Centralized Coordination**
```c
pthread_mutex_t coordinator = PTHREAD_MUTEX_INITIALIZER;

void coordinated_access() {
    // Single point of coordination prevents livelock
    pthread_mutex_lock(&coordinator);
    access_resources();
    pthread_mutex_unlock(&coordinator);
}
```

## Comparison Summary

```
Deadlock vs Livelock:

Deadlock:
  State: Blocked
  CPU: Idle
  Solution: Break circular wait
  Detection: Thread dumps show waiting

Livelock:
  State: Active
  CPU: Busy
  Solution: Add randomness/priority
  Detection: High CPU, no progress
```

## Exercises

### Exercise 1: Identify Livelock
Find the livelock in this code:
```c
void* worker(void* arg) {
    while (!try_acquire_resources()) {
        yield_to_others();
    }
    do_work();
}
```

### Exercise 2: Fix Network Collision
Implement proper exponential backoff for network transmission simulation.

### Exercise 3: Build Progress Monitor
Create a monitoring system that detects livelock conditions.

## Summary

**Livelock** is threads being active but not making progress due to:
- Synchronized retry patterns
- Excessive politeness
- Lack of randomization
- Collision without proper backoff

**Key Differences from Deadlock:**
- Threads are active (not blocked)
- High CPU usage
- Harder to detect
- Different solutions needed

**Prevention:**
- Random/exponential backoff
- Priority schemes
- Lock ordering
- Progress monitoring

## Further Reading

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- Ethernet CSMA/CD specification (IEEE 802.3)

## Next Topic

Continue to [04-starvation.md](./04-starvation.md) to learn about starvation.
