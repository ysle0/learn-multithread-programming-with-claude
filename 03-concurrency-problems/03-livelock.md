# Livelock

## Livelock이란?

**Livelock**은 thread들이 (deadlock과 달리) 차단되지 않지만, 서로에게 반응하여 상태를 계속 변경하면서도 의미 있는 진전을 이루지 못하는 상황입니다. Thread들은 활성 상태를 유지하며 CPU 자원을 소비하지만, 시스템 전체적으로는 완료를 향해 나아가지 못합니다.

좁은 복도에서 두 사람이 서로 지나가려고 하는 상황을 떠올려 보세요. 두 사람이 동시에 같은 방향으로 비키고, 다시 반대 방향으로 비키기를 반복하면서 결국 지나가지 못하는 것과 같습니다.

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

### 시각적 비교

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

## 고전적인 예시: 복도 문제

```
Person A ←─────────────────→ Person B
         Narrow Hallway

Step 1: A moves left, B moves left   (both still blocked)
Step 2: A moves right, B moves right (both still blocked)
Step 3: A moves left, B moves left   (both still blocked)
...repeats forever...
```

### 코드 구현

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

        // 초기화하고 다시 시도
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

        // 초기화하고 다시 시도
        person_b.trying_left = false;
        person_b.trying_right = false;
    }
    return NULL;
}

// LIVELOCK 발생 - 둘 다 계속 움직이지만 결코 지나가지 못함!
```

## 일반적인 Livelock 패턴

### 패턴 1: 충돌 회피

Thread들이 충돌을 감지하고 물러나지만, 동기화된 방식으로 수행하는 경우입니다.

```c
#include <pthread.h>
#include <stdbool.h>
#include <stdio.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_livelock(void* arg) {
    int id = *(int*)arg;

    while (true) {
        // 두 리소스 모두 획득 시도
        pthread_mutex_lock(&resource_a);

        if (pthread_mutex_trylock(&resource_b) != 0) {
            // B 획득 실패, A를 해제하고 재시도
            printf("Thread %d: Failed to get B, releasing A\n", id);
            pthread_mutex_unlock(&resource_a);

            // 문제: 두 thread가 동시에 이 작업을 수행!
            // 해제와 재시도를 영원히 반복
            continue;
        }

        // 임계 영역
        printf("Thread %d: Got both resources!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}

// LIVELOCK: 두 thread가 동시에 재시도하면 반복적으로 충돌
```

**타임라인:**
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

### 패턴 2: 양보하는 Thread

Thread들이 다른 thread에게 "양보"하려 하지만, 모두 동시에 양보하는 경우입니다.

```c
#include <pthread.h>
#include <stdbool.h>
#include <sched.h>

volatile bool thread1_wants = false;
volatile bool thread2_wants = false;

void* polite_thread1(void* arg) {
    while (true) {
        thread1_wants = true;

        // 양보: 다른 thread가 원하면 양보
        while (thread2_wants) {
            thread1_wants = false;  // 길을 비켜줌
            sched_yield();          // 다른 thread에게 양보
            thread1_wants = true;   // 다시 원함
        }

        // 임계 영역
        critical_section();

        thread1_wants = false;
    }
    return NULL;
}

void* polite_thread2(void* arg) {
    while (true) {
        thread2_wants = true;

        // 양보: 다른 thread가 원하면 양보
        while (thread1_wants) {
            thread2_wants = false;  // 길을 비켜줌
            sched_yield();          // 다른 thread에게 양보
            thread2_wants = true;   // 다시 원함
        }

        // 임계 영역
        critical_section();

        thread2_wants = false;
    }
    return NULL;
}

// LIVELOCK: 서로에게 계속 양보!
```

### 패턴 3: 메시지 재전송

분산 시스템에서 노드들이 충돌 시 재전송하지만 더 많은 충돌을 만드는 경우입니다.

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
    // 충돌 감지 시뮬레이션
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
        // 고정 재시도 간격 - 동기화된 재시도 유발
        usleep(1000);  // 항상 1ms 대기

        // LIVELOCK: 모든 노드가 같은 시간에 재시도!
    }
}
```

## 해결 방법 및 예방

### 해결 방법 1: 랜덤 Backoff

무작위성을 도입하여 동기화를 깨뜨립니다.

```c
#include <pthread.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_random_backoff(void* arg) {
    int id = *(int*)arg;
    srand(time(NULL) + id);  // thread마다 다른 시드

    while (true) {
        pthread_mutex_lock(&resource_a);

        if (pthread_mutex_trylock(&resource_b) != 0) {
            pthread_mutex_unlock(&resource_a);

            // 랜덤 backoff: 0-10ms
            int backoff = rand() % 10000;
            printf("Thread %d: Backing off %dμs\n", id, backoff);
            usleep(backoff);
            continue;
        }

        // 임계 영역
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

### 해결 방법 2: Exponential Backoff

재시도할 때마다 backoff 시간을 증가시킵니다 (이더넷 CSMA/CD와 유사).

```c
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>

#define MAX_BACKOFF 1000000  // 1 second

void* thread_with_exponential_backoff(void* arg) {
    int id = *(int*)arg;
    int backoff = 1000;  // 1ms부터 시작

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

        // 임계 영역
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

### 해결 방법 3: 우선순위 기반 해결

한 thread에 더 높은 우선순위를 부여합니다.

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

        // 높은 우선순위 thread에게 양보
        while (high_priority_wants) {
            low_priority_wants = false;
            sched_yield();
            low_priority_wants = true;
        }

        // 임계 영역
        critical_section();
        low_priority_wants = false;
    }
    return NULL;
}

void* high_priority_thread(void* arg) {
    while (true) {
        high_priority_wants = true;

        // 양보하지 않음 - 우선권 행사!
        // 임계 영역
        critical_section();
        high_priority_wants = false;
    }
    return NULL;
}

// LIVELOCK 없음: 높은 우선순위가 항상 진행
```

### 해결 방법 4: Lock 순서 지정

일관된 lock 순서를 사용하여 재시도를 방지합니다.

```c
#include <pthread.h>

pthread_mutex_t resource_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t resource_b = PTHREAD_MUTEX_INITIALIZER;

void* thread_with_ordering(void* arg) {
    // 항상 순서대로 획득: A 다음 B
    pthread_mutex_lock(&resource_a);
    pthread_mutex_lock(&resource_b);

    // 임계 영역
    critical_section();

    pthread_mutex_unlock(&resource_b);
    pthread_mutex_unlock(&resource_a);

    return NULL;
}

// LIVELOCK 없음: trylock 없음, 재시도 불필요
```

### 해결 방법 5: Timeout과 랜덤화 결합

timeout과 랜덤 재시도를 결합합니다.

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
        timeout.tv_sec += 1;  // 1초 timeout

        pthread_mutex_lock(&resource_a);

        int result = pthread_mutex_timedlock(&resource_b, &timeout);

        if (result == ETIMEDOUT) {
            pthread_mutex_unlock(&resource_a);

            // 재시도 전 랜덤 backoff
            usleep(rand() % 100000);
            continue;
        }

        // 임계 영역
        printf("Thread %d: Success!\n", id);
        pthread_mutex_unlock(&resource_b);
        pthread_mutex_unlock(&resource_a);
        break;
    }

    return NULL;
}
```

## 실제 사례

### 예시 1: 네트워크 충돌 (이더넷)

```c
// 간소화된 CSMA/CD (Carrier Sense Multiple Access with Collision Detection)
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
        // 캐리어 감지
        if (channel_busy()) {
            wait_until_idle();
        }

        // 전송
        if (send_frame()) {
            printf("Node %d: Transmission successful\n", node->id);
            return;
        }

        // 충돌 감지
        node->collisions++;
        printf("Node %d: Collision #%d\n", node->id, node->collisions);

        // 이진 지수 backoff
        int k = (attempt < 10) ? attempt : 10;
        int backoff_slots = rand() % (1 << k);  // 0 ~ 2^k - 1
        usleep(backoff_slots * 512);  // 슬롯당 512μs

        attempt++;
    }

    printf("Node %d: Failed after %d attempts\n", node->id, MAX_ATTEMPTS);
}
```

### 예시 2: 데이터베이스 재시도 로직

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
            // Deadlock 회피가 livelock을 유발!
            pthread_mutex_unlock(&records[id1].mutex);
            attempts++;
            continue;  // 고정 재시도 = LIVELOCK
        }

        // 두 레코드 갱신
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;  // 실패
}

bool update_records_fixed(int id1, int id2, int delta) {
    int attempts = 0;

    while (attempts < 100) {
        pthread_mutex_lock(&records[id1].mutex);

        if (pthread_mutex_trylock(&records[id2].mutex) != 0) {
            pthread_mutex_unlock(&records[id1].mutex);

            // 랜덤 backoff로 livelock 방지
            usleep(rand() % 10000);
            attempts++;
            continue;
        }

        // 두 레코드 갱신
        records[id1].value -= delta;
        records[id2].value += delta;

        pthread_mutex_unlock(&records[id2].mutex);
        pthread_mutex_unlock(&records[id1].mutex);
        return true;
    }

    return false;
}
```

### 예시 3: 분산 합의

```c
#include <stdio.h>
#include <stdbool.h>
#include <stdlib.h>

typedef struct {
    int id;
    int proposed_value;
    int seen_proposals;
} Node;

// livelock 가능성이 있는 간소화된 합의
void reach_consensus_bad(Node* nodes, int num_nodes) {
    bool consensus_reached = false;

    while (!consensus_reached) {
        // 각 노드가 자신의 값을 제안
        for (int i = 0; i < num_nodes; i++) {
            nodes[i].seen_proposals = 0;

            // 다른 노드의 제안 확인
            for (int j = 0; j < num_nodes; j++) {
                if (nodes[j].proposed_value == nodes[i].proposed_value) {
                    nodes[i].seen_proposals++;
                }
            }

            // 과반수가 아니면 제안 변경
            if (nodes[i].seen_proposals < num_nodes / 2) {
                // 새로운 랜덤 값 선택
                nodes[i].proposed_value = rand() % 100;
                printf("Node %d: Changing proposal\n", i);
            }
        }

        // 합의 확인
        int first_value = nodes[0].proposed_value;
        consensus_reached = true;
        for (int i = 1; i < num_nodes; i++) {
            if (nodes[i].proposed_value != first_value) {
                consensus_reached = false;
                break;
            }
        }
    }

    // LIVELOCK: 노드들이 계속 제안을 변경!
}

// 리더 선출을 통한 수정 버전
void reach_consensus_good(Node* nodes, int num_nodes) {
    // 리더 선출 (예: 가장 낮은 ID)
    int leader_id = 0;
    for (int i = 1; i < num_nodes; i++) {
        if (nodes[i].id < nodes[leader_id].id) {
            leader_id = i;
        }
    }

    // 모두 리더의 제안을 수용
    int consensus_value = nodes[leader_id].proposed_value;
    for (int i = 0; i < num_nodes; i++) {
        nodes[i].proposed_value = consensus_value;
    }

    printf("Consensus reached: %d\n", consensus_value);
    // LIVELOCK 없음: 단일 의사결정자
}
```

## 내부 메커니즘

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

## 탐지 전략

### 1. 진행 상황 모니터링

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

### 2. 재시도 카운터

```c
#define MAX_RETRIES 1000

int retry_count = 0;

void detect_excessive_retries() {
    retry_count++;

    if (retry_count > MAX_RETRIES) {
        printf("LIVELOCK suspected: %d retries!\n", retry_count);
        // 교정 조치 수행
        abort();
    }
}
```

### 3. CPU 사용률 분석

```bash
# CPU 사용률 모니터링
top -H -p <pid>

# thread들이 높은 CPU를 보이지만 진전이 없으면 → livelock

# perf를 사용하여 thread들이 무엇을 하는지 확인
perf record -p <pid> -g
perf report
```

## 예방 모범 사례

### 체크리스트

- [ ] 고정 지연 대신 랜덤 backoff 사용
- [ ] 재시도에 exponential backoff 구현
- [ ] 최대 재시도 횟수 제한 설정
- [ ] 가능하면 trylock 대신 lock 순서 지정 사용
- [ ] timeout 메커니즘 추가
- [ ] 진행 상황 지표 모니터링
- [ ] 부하 상태에서 다중 thread로 테스트
- [ ] 대칭적 재시도 로직 회피

### 설계 패턴

**패턴 1: 비대칭 동작**
```c
void* thread_function(void* arg) {
    int id = *(int*)arg;

    // 짝수 thread는 전략 A 사용
    if (id % 2 == 0) {
        strategy_a();
    }
    // 홀수 thread는 전략 B 사용
    else {
        strategy_b();
    }
}
```

**패턴 2: 중앙 집중식 조정**
```c
pthread_mutex_t coordinator = PTHREAD_MUTEX_INITIALIZER;

void coordinated_access() {
    // 단일 조정 지점으로 livelock 방지
    pthread_mutex_lock(&coordinator);
    access_resources();
    pthread_mutex_unlock(&coordinator);
}
```

## 비교 요약

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

## 연습 문제

### 연습 문제 1: Livelock 식별
이 코드에서 livelock을 찾으세요:
```c
void* worker(void* arg) {
    while (!try_acquire_resources()) {
        yield_to_others();
    }
    do_work();
}
```

### 연습 문제 2: 네트워크 충돌 수정
네트워크 전송 시뮬레이션에 적절한 exponential backoff를 구현하세요.

### 연습 문제 3: 진행 상황 모니터 구축
Livelock 상태를 감지하는 모니터링 시스템을 만드세요.

## 요약

**Livelock**은 다음과 같은 이유로 thread가 활성 상태이지만 진전을 이루지 못하는 현상입니다:
- 동기화된 재시도 패턴
- 과도한 양보
- 무작위성 부족
- 적절한 backoff 없는 충돌

**Deadlock과의 주요 차이점:**
- Thread가 활성 상태 (차단되지 않음)
- 높은 CPU 사용률
- 탐지가 더 어려움
- 다른 해결 방법 필요

**예방 방법:**
- 랜덤/exponential backoff
- 우선순위 체계
- Lock 순서 지정
- 진행 상황 모니터링

## 추가 자료

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- 이더넷 CSMA/CD 명세 (IEEE 802.3)

## 다음 주제

[04-starvation.md](./04-starvation.md)에서 starvation에 대해 알아보세요.
