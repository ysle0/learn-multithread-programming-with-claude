# Priority Inversion

## What is Priority Inversion?

**Priority Inversion** is a situation where a high-priority thread is indirectly blocked by a low-priority thread, violating the priority-based scheduling guarantees. This occurs when a low-priority thread holds a resource that a high-priority thread needs, while a medium-priority thread preempts the low-priority thread, preventing it from releasing the resource.

### The Problem in Simple Terms

```
High Priority Thread:    "I need that resource NOW!"
                         ↓ (blocked)
Low Priority Thread:     "I have it, but I'm not running..."
                         ↓ (preempted)
Medium Priority Thread:  "I'm running instead!"
                         ↓ (prevents progress)

Result: High priority waits for medium priority (WRONG!)
```

## Visual Representation

### Priority Inversion Scenario

```
Timeline:
────────────────────────────────────────────────────────────▶

Priority Levels:
  High   (H) : ─────┐      [BLOCKED]          [RUNS]
                     │          ⏳                ✓
  Medium (M) : ─────┼───────────────[RUNS]──────┐
                     │                            │
  Low    (L) : [RUNS]──────[PREEMPTED]───────[RUNS][RELEASE]
               Lock R                              Unlock R

  Problem: H waits for L, but L can't run because M is running!
           High priority effectively has LOWER priority than medium!
```

### Resource Dependency Graph

```
┌──────────────┐
│ High-Priority│──┐
│   Thread H   │  │ needs
└──────────────┘  │
                  ↓
              ┌────────┐
              │Resource│ held by
              │   R    │←────────┐
              └────────┘         │
                                 │
                          ┌──────────────┐
                          │ Low-Priority │
                          │   Thread L   │
                          └──────────────┘
                                 ↑
                                 │ preempted by
                                 │
                          ┌──────────────┐
                          │Medium-Priority│
                          │   Thread M   │
                          └──────────────┘

H waits for L, but L can't run → PRIORITY INVERSION
```

## The Mars Pathfinder Incident (1997)

### Background

The Mars Pathfinder landed on Mars on July 4, 1997. Shortly after landing, the spacecraft began experiencing system resets, causing data loss and mission delays.

### The Problem

```
Thread Priorities:
  High:   ASI/MET Bus Management Task (critical communication)
  Medium: Communications Task
  Low:    Meteorological Data Task

Shared Resource: Information Bus (protected by mutex)
```

### What Happened

```
Timeline of the Bug:

1. Low-priority thread (Weather) locks the information bus
   [L] ━━━ Lock Bus ━━━

2. Low-priority thread is preempted by medium-priority thread
   [L] ━━━ [PREEMPTED]
   [M] ━━━━━━━━━ Running ━━━━━━━━━

3. High-priority thread (Bus Management) wakes up and needs bus
   [H] ━━━ [BLOCKED on bus] ━━━━━━━
   [M] ━━━━━━━━━ Still Running ━━━
   [L] ━━━ [Still Preempted]

4. High-priority thread doesn't get CPU for too long
   Watchdog timer expires → SYSTEM RESET

Result: Mars Pathfinder kept resetting!
```

### The Fix

NASA engineers (with help from VxWorks experts) enabled **priority inheritance** in the VxWorks mutex implementation.

```c
// Before: Normal mutex (no priority inheritance)
semaphore = semMCreate(SEM_Q_PRIORITY);

// After: Mutex with priority inheritance
semaphore = semMCreate(SEM_Q_PRIORITY | SEM_INVERSION_SAFE);
```

**How it helped:**
```
With Priority Inheritance:

1. Low-priority thread locks bus
   [L] ━━━ Lock Bus ━━━ (priority = LOW)

2. High-priority thread blocks on bus
   [L] ━━━ (priority BOOSTED to HIGH!) ━━━
   [H] ━━━ [BLOCKED]

3. Low-priority thread (now running at HIGH priority) completes
   [L] ━━━ Finish & Unlock ━━━
   [M] doesn't preempt (L now has higher priority!)

4. High-priority thread acquires bus and runs
   [H] ━━━ Lock & Run ━━━ ✓

Result: NO system resets!
```

## Types of Priority Inversion

### 1. Bounded Priority Inversion

Duration is limited by the critical section of the low-priority thread.

```
Max delay = Length of L's critical section

Timeline:
  H: ───[BLOCKED]──── (bounded wait)
  L: ──[CRITICAL]──── (finishes quickly)
```

### 2. Unbounded Priority Inversion

Duration is extended by medium-priority threads.

```
Max delay = Unknown (depends on M's execution time)

Timeline:
  H: ───[BLOCKED]──────────────────── (unbounded wait!)
  M: ───────[RUNNING]─────────────────
  L: ──[CRITICAL]──[PREEMPTED]────────
```

## Code Examples

### Example 1: Demonstrating Priority Inversion

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <sched.h>

pthread_mutex_t resource = PTHREAD_MUTEX_INITIALIZER;

void set_thread_priority(int priority) {
    struct sched_param param;
    param.sched_priority = priority;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);
}

void* low_priority_thread(void* arg) {
    set_thread_priority(1);  // Lowest priority
    printf("[L] Starting\n");

    pthread_mutex_lock(&resource);
    printf("[L] Acquired resource\n");

    // Simulate work with resource
    printf("[L] Working with resource (5 seconds)...\n");
    sleep(5);

    printf("[L] Releasing resource\n");
    pthread_mutex_unlock(&resource);

    return NULL;
}

void* medium_priority_thread(void* arg) {
    set_thread_priority(50);  // Medium priority
    sleep(1);  // Let low priority thread acquire lock first

    printf("[M] Starting - will preempt low priority!\n");

    // Compute-intensive work (no shared resources)
    printf("[M] Doing work (prevents low priority from finishing)...\n");
    for (volatile long i = 0; i < 1000000000L; i++);

    printf("[M] Finished\n");
    return NULL;
}

void* high_priority_thread(void* arg) {
    set_thread_priority(99);  // Highest priority
    sleep(2);  // Let low priority acquire lock, medium preempt

    printf("[H] Starting - NEED RESOURCE!\n");

    pthread_mutex_lock(&resource);
    printf("[H] Finally acquired resource (DELAYED by medium!)\n");

    // Critical work
    printf("[H] Working\n");

    pthread_mutex_unlock(&resource);
    printf("[H] Done\n");

    return NULL;
}

int main() {
    pthread_t low, medium, high;

    printf("=== Demonstrating Priority Inversion ===\n");

    pthread_create(&low, NULL, low_priority_thread, NULL);
    pthread_create(&medium, NULL, medium_priority_thread, NULL);
    pthread_create(&high, NULL, high_priority_thread, NULL);

    pthread_join(low, NULL);
    pthread_join(medium, NULL);
    pthread_join(high, NULL);

    return 0;
}

// Expected output shows H waiting for M to finish,
// even though M doesn't use the resource!
```

### Example 2: Priority Inheritance Solution

```c
#include <pthread.h>
#include <stdio.h>

typedef struct {
    pthread_mutex_t mutex;
    pthread_t owner;
    int owner_original_priority;
    int inherited_priority;
} PriorityInheritanceMutex;

void pi_mutex_init(PriorityInheritanceMutex* pim) {
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);

    // Enable priority inheritance protocol
    pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);

    pthread_mutex_init(&pim->mutex, &attr);
    pthread_mutexattr_destroy(&attr);

    pim->owner = 0;
    pim->owner_original_priority = 0;
    pim->inherited_priority = 0;
}

void pi_mutex_lock(PriorityInheritanceMutex* pim) {
    pthread_t self = pthread_self();
    struct sched_param my_param;
    int my_policy;
    pthread_getschedparam(self, &my_policy, &my_param);

    // Try to acquire
    if (pthread_mutex_trylock(&pim->mutex) == 0) {
        // Got it immediately
        pim->owner = self;
        pim->owner_original_priority = my_param.sched_priority;
        return;
    }

    // Someone else owns it - boost their priority
    if (pim->owner) {
        struct sched_param owner_param;
        int owner_policy;
        pthread_getschedparam(pim->owner, &owner_policy, &owner_param);

        if (my_param.sched_priority > owner_param.sched_priority) {
            // Boost owner's priority to our level
            owner_param.sched_priority = my_param.sched_priority;
            pthread_setschedparam(pim->owner, owner_policy, &owner_param);

            printf("Priority inheritance: Boosted owner to priority %d\n",
                   my_param.sched_priority);
        }
    }

    // Now wait for lock
    pthread_mutex_lock(&pim->mutex);
    pim->owner = self;
    pim->owner_original_priority = my_param.sched_priority;
}

void pi_mutex_unlock(PriorityInheritanceMutex* pim) {
    pthread_t self = pthread_self();

    if (pim->owner == self) {
        // Restore original priority if it was boosted
        struct sched_param param;
        int policy;
        pthread_getschedparam(self, &policy, &param);

        if (param.sched_priority != pim->owner_original_priority) {
            param.sched_priority = pim->owner_original_priority;
            pthread_setschedparam(self, policy, &param);

            printf("Priority restored to %d\n", pim->owner_original_priority);
        }

        pim->owner = 0;
    }

    pthread_mutex_unlock(&pim->mutex);
}

// This implementation automatically boosts the priority
// of the lock holder when a higher priority thread waits
```

### Example 3: Priority Ceiling Protocol

An alternative to priority inheritance.

```c
#include <pthread.h>
#include <stdio.h>

typedef struct {
    pthread_mutex_t mutex;
    int ceiling_priority;
} PriorityCeilingMutex;

void pc_mutex_init(PriorityCeilingMutex* pcm, int ceiling) {
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);

    // Set priority ceiling protocol
    pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_PROTECT);
    pthread_mutexattr_setprioceiling(&attr, ceiling);

    pthread_mutex_init(&pcm->mutex, &attr);
    pthread_mutexattr_destroy(&attr);

    pcm->ceiling_priority = ceiling;
}

void pc_mutex_lock(PriorityCeilingMutex* pcm) {
    // Thread automatically runs at ceiling priority while holding lock
    pthread_mutex_lock(&pcm->mutex);

    printf("Lock acquired - running at ceiling priority %d\n",
           pcm->ceiling_priority);
}

void pc_mutex_unlock(PriorityCeilingMutex* pcm) {
    printf("Lock released - priority restored\n");
    pthread_mutex_unlock(&pcm->mutex);
}

// With priority ceiling:
// - Lock holder always runs at ceiling priority
// - No medium-priority thread can preempt
// - Prevents priority inversion entirely!
```

## Comparison of Solutions

### Priority Inheritance

```
Advantages:
  + No need to know priorities in advance
  + Priority only raised when needed
  + Works with dynamic priorities

Disadvantages:
  - More complex implementation
  - Can lead to deadlock in chains
  - Runtime overhead for priority changes
```

### Priority Ceiling

```
Advantages:
  + Simpler implementation
  + Prevents deadlock
  + Predictable behavior

Disadvantages:
  - Need to know all priorities in advance
  - May raise priority unnecessarily
  - Less flexible
```

### Comparison Table

```
┌──────────────────┬──────────────────┬───────────────────┐
│    Feature       │   Inheritance    │     Ceiling       │
├──────────────────┼──────────────────┼───────────────────┤
│ Priority Change  │  On demand       │  Always           │
│ Deadlock Risk    │  Possible        │  No (if correct)  │
│ Setup Complexity │  Low             │  High             │
│ Runtime Overhead │  Medium          │  Low              │
│ Predictability   │  Lower           │  Higher           │
└──────────────────┴──────────────────┴───────────────────┘
```

## Real-World Examples

### Example 1: Real-Time Control System

```c
// Aircraft flight control system
#define PRIORITY_CRITICAL   99  // Flight controls
#define PRIORITY_HIGH       80  // Navigation
#define PRIORITY_MEDIUM     50  // Communications
#define PRIORITY_LOW        20  // Logging

pthread_mutex_t sensor_data_mutex;

void* critical_flight_control(void* arg) {
    set_priority(PRIORITY_CRITICAL);

    while (1) {
        // MUST run every 10ms
        pthread_mutex_lock(&sensor_data_mutex);
        read_sensors();
        update_flight_controls();
        pthread_mutex_unlock(&sensor_data_mutex);

        usleep(10000);  // 10ms period
    }
}

void* medium_comms_task(void* arg) {
    set_priority(PRIORITY_MEDIUM);

    while (1) {
        // Shouldn't delay critical tasks!
        send_telemetry();
        sleep(1);
    }
}

void* low_logging_task(void* arg) {
    set_priority(PRIORITY_LOW);

    while (1) {
        pthread_mutex_lock(&sensor_data_mutex);
        log_sensor_data();  // If this is preempted...
        pthread_mutex_unlock(&sensor_data_mutex);

        sleep(1);
    }
}

// Without priority inheritance:
// - Logging locks sensor_data_mutex
// - Comms task preempts logging
// - Flight control MISSES DEADLINE!
// Result: CATASTROPHIC

// With priority inheritance:
// - Logging boosted to CRITICAL while holding lock
// - Comms cannot preempt
// - Flight control meets deadline
// Result: SAFE
```

### Example 2: Industrial Robot Control

```c
// Robot with multiple control loops
typedef struct {
    PriorityInheritanceMutex position_mutex;
    double x, y, z;
} RobotPosition;

RobotPosition robot_pos;

void* servo_control(void* arg) {
    // Runs at 1kHz - MUST be fast
    set_priority(PRIORITY_REALTIME);

    while (1) {
        pi_mutex_lock(&robot_pos.position_mutex);

        // Update servo positions
        update_servos(robot_pos.x, robot_pos.y, robot_pos.z);

        pi_mutex_unlock(&robot_pos.position_mutex);

        usleep(1000);  // 1ms period
    }
}

void* path_planning(void* arg) {
    // Medium priority
    set_priority(PRIORITY_NORMAL);

    while (1) {
        // Compute next position
        calculate_path();
        usleep(10000);  // 10ms period
    }
}

void* ui_update(void* arg) {
    // Low priority
    set_priority(PRIORITY_LOW);

    while (1) {
        pi_mutex_lock(&robot_pos.position_mutex);

        // Update display
        display_position(robot_pos.x, robot_pos.y, robot_pos.z);

        pi_mutex_unlock(&robot_pos.position_mutex);

        usleep(100000);  // 100ms period
    }
}

// Priority inheritance ensures:
// - UI never delays servo control
// - System remains responsive
// - Real-time deadlines are met
```

### Example 3: Medical Device

```c
// Insulin pump controller
#define PRIORITY_SAFETY      99  // Safety monitoring
#define PRIORITY_DELIVERY    80  // Insulin delivery
#define PRIORITY_UI          20  // User interface

pthread_mutex_t dose_calculation_mutex;

void* safety_monitor(void* arg) {
    set_priority(PRIORITY_SAFETY);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);

        // Check for unsafe conditions
        if (blood_glucose_too_low() || pump_malfunction()) {
            emergency_stop();
        }

        pthread_mutex_unlock(&dose_calculation_mutex);

        usleep(100000);  // Check every 100ms
    }
}

void* insulin_delivery(void* arg) {
    set_priority(PRIORITY_DELIVERY);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);
        calculate_and_deliver_dose();
        pthread_mutex_unlock(&dose_calculation_mutex);

        sleep(5);  // Every 5 minutes
    }
}

void* user_interface(void* arg) {
    set_priority(PRIORITY_UI);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);
        update_display();
        pthread_mutex_unlock(&dose_calculation_mutex);

        usleep(500000);  // Update every 500ms
    }
}

// CRITICAL: UI must not delay safety monitoring
// Priority inheritance is ESSENTIAL for patient safety
```

## Internal Mechanisms

### Linux 커널의 rt_mutex와 우선순위 상속

Linux 커널은 rt_mutex를 통해 우선순위 상속(Priority Inheritance)을 구현합니다.

#### rt_mutex 구조체

```c
// linux/include/linux/rtmutex.h
struct rt_mutex {
    raw_spinlock_t      wait_lock;      // 대기자 리스트 보호
    struct rb_root_cached waiters;       // 우선순위 정렬된 대기자들
    struct task_struct  *owner;          // 현재 소유자

    // PI chain 추적용
    struct rt_mutex_waiter *top_waiter;  // 가장 높은 우선순위 대기자
};

struct rt_mutex_waiter {
    struct rb_node          tree_entry;     // RB 트리 노드
    struct rb_node          pi_tree_entry;  // PI 트리 노드
    struct task_struct      *task;          // 대기 중인 태스크
    struct rt_mutex         *lock;          // 대기 중인 락
    int                     prio;           // 대기자 우선순위
    u64                     deadline;       // SCHED_DEADLINE용
};
```

#### 우선순위 상속 체인 (PI Chain)

```
PI Chain 예시:

  High (prio=99)         Medium (prio=50)        Low (prio=10)
  ┌──────────┐           ┌──────────┐           ┌──────────┐
  │ Thread H │           │ Thread M │           │ Thread L │
  │ prio: 99 │           │ prio: 50 │           │ prio: 10 │
  └────┬─────┘           └────┬─────┘           └────┬─────┘
       │                      │                      │
       │ waits for            │ waits for            │ owns
       ▼                      ▼                      ▼
  ┌─────────┐            ┌─────────┐            ┌─────────┐
  │ Mutex A │───────────►│ Mutex B │───────────►│ Mutex C │
  │ owner:M │            │ owner:L │            │ owner:L │
  └─────────┘            └─────────┘            └─────────┘

  PI Chain: H → A → M → B → L

  우선순위 전파:
    L의 effective priority = max(10, 50, 99) = 99
    M의 effective priority = max(50, 99) = 99 (중간 노드)
```

#### 커널 PI 구현 (간략화)

```c
// linux/kernel/locking/rtmutex.c

static int rt_mutex_adjust_prio_chain(struct task_struct *task,
                                      int deadlock_detect,
                                      struct rt_mutex *orig_lock,
                                      struct rt_mutex *next_lock,
                                      struct rt_mutex_waiter *orig_waiter) {
    struct rt_mutex_waiter *waiter, *top_waiter;
    struct rt_mutex *lock;
    struct task_struct *next;
    int ret = 0;

    // PI chain 순회
    for (;;) {
        // 현재 태스크가 대기 중인 락 확인
        waiter = task->pi_blocked_on;
        if (!waiter)
            break;  // chain 끝

        // 대기 중인 락
        lock = waiter->lock;

        // 락 소유자
        next = rt_mutex_owner(lock);
        if (!next)
            break;  // 소유자 없음

        // 우선순위 상속 필요 여부 확인
        if (waiter->prio <= next->normal_prio) {
            // 대기자 우선순위가 더 높음
            // → 소유자 우선순위 상승

            // effective priority 업데이트
            rt_mutex_setprio(next, waiter->prio);

            // 소유자의 PI waiter 리스트 업데이트
            rt_mutex_enqueue_pi(next, waiter);
        }

        // 교착 탐지
        if (deadlock_detect && next == current) {
            ret = -EDEADLK;
            break;
        }

        // chain 다음 노드로
        task = next;

        // 최대 깊이 제한 (무한 루프 방지)
        if (++chain_depth > MAX_CHAIN_DEPTH) {
            ret = -EDEADLK;  // 너무 긴 chain
            break;
        }
    }

    return ret;
}

// 우선순위 설정
static void rt_mutex_setprio(struct task_struct *p, int prio) {
    struct rq *rq;

    rq = task_rq_lock(p);

    // effective priority 저장
    p->prio = prio;

    // 스케줄러에게 알림
    if (task_on_rq_queued(p)) {
        dequeue_task(rq, p, DEQUEUE_SAVE);
        enqueue_task(rq, p, ENQUEUE_RESTORE);
    }

    // 필요시 선점
    check_preempt_curr(rq, p);

    task_rq_unlock(rq);
}
```

### FUTEX_LOCK_PI 시스템 콜

사용자 공간에서 우선순위 상속을 사용하는 방법입니다.

```c
// linux/kernel/futex.c

static int futex_lock_pi(u32 __user *uaddr, int fshared,
                        ktime_t *time, int trylock) {
    struct futex_hash_bucket *hb;
    struct futex_q q;
    struct rt_mutex_waiter rt_waiter;
    struct task_struct *owner;
    int ret;

    // 1. Fast path: userspace에서 lock 시도
    // 값이 0이면 현재 TID로 설정
    ret = futex_trylock_pi(uaddr);
    if (ret == 0)
        return 0;  // 즉시 획득 성공

    // 2. Slow path: 커널 진입
    hb = futex_hash(&q.key);
    spin_lock(&hb->lock);

    // 현재 소유자 확인 (futex 값 = owner TID)
    owner = futex_find_owner(uaddr);
    if (!owner) {
        // 소유자를 찾을 수 없음
        ret = -EINVAL;
        goto out_unlock;
    }

    // 3. rt_mutex 연결 및 PI 설정
    // futex에 연결된 rt_mutex를 통해 PI chain 구축
    ret = rt_mutex_start_proxy_lock(&q.pi_state->pi_mutex,
                                    &rt_waiter, current);

    if (ret) {
        // 이미 chain에 있음 (교착 상태 가능)
        goto out_unlock;
    }

    // 4. 대기
    spin_unlock(&hb->lock);

    ret = rt_mutex_wait_proxy_lock(&q.pi_state->pi_mutex,
                                   time, &rt_waiter);

    // 5. 깨어남: 락 획득 완료
    return ret;

out_unlock:
    spin_unlock(&hb->lock);
    return ret;
}

// Futex 값 형식 (PI mode)
/*
 * Bit 31: FUTEX_WAITERS (대기자 있음)
 * Bit 30: FUTEX_OWNER_DIED (소유자 사망)
 * Bit 0-29: Owner TID
 *
 * 예: 0x80001234 = TID 0x1234가 소유, 대기자 있음
 */
```

### PTHREAD_PRIO_INHERIT 구현

```c
// glibc pthread mutex with priority inheritance
// nptl/pthread_mutex_lock.c (개념적)

int __pthread_mutex_lock_pi(pthread_mutex_t *mutex) {
    int kind = mutex->__data.__kind;
    pid_t tid = THREAD_GETMEM(THREAD_SELF, tid);

    // Fast path: 비경합 상태
    if (atomic_compare_exchange_weak(&mutex->__data.__lock,
                                     0, tid)) {
        return 0;  // 즉시 획득
    }

    // Slow path: 경합 상태 - PI futex 사용
    int oldval = atomic_load(&mutex->__data.__lock);

    while (1) {
        // FUTEX_WAITERS 비트 설정
        int newval = oldval | FUTEX_WAITERS;

        if (oldval != newval) {
            if (!atomic_compare_exchange_weak(&mutex->__data.__lock,
                                              &oldval, newval))
                continue;
        }

        // PI futex 대기
        // → 커널이 소유자 우선순위를 자동으로 상승
        int ret = syscall(SYS_futex, &mutex->__data.__lock,
                         FUTEX_LOCK_PI, 0, NULL, NULL, 0);

        if (ret == 0) {
            // 획득 성공
            return 0;
        }

        if (ret != -EAGAIN)
            return ret;

        // 재시도
        oldval = atomic_load(&mutex->__data.__lock);
    }
}

int __pthread_mutex_unlock_pi(pthread_mutex_t *mutex) {
    pid_t tid = THREAD_GETMEM(THREAD_SELF, tid);
    int oldval = atomic_load(&mutex->__data.__lock);

    // 대기자 확인
    if (!(oldval & FUTEX_WAITERS)) {
        // 대기자 없음: userspace unlock
        if (atomic_compare_exchange_strong(&mutex->__data.__lock,
                                           &oldval, 0))
            return 0;
    }

    // 대기자 있음: 커널 호출하여 PI 정리 및 wake
    return syscall(SYS_futex, &mutex->__data.__lock,
                   FUTEX_UNLOCK_PI, 0, NULL, NULL, 0);
}
```

### Priority Ceiling 프로토콜

```c
// PTHREAD_PRIO_PROTECT 구현 원리

typedef struct {
    pthread_mutex_t mutex;
    int ceiling;          // 최대 우선순위
    int saved_priority;   // 원래 우선순위 저장
} ceiling_mutex_t;

int ceiling_mutex_lock(ceiling_mutex_t *cm) {
    // 현재 우선순위 저장
    struct sched_param param;
    int policy;
    pthread_getschedparam(pthread_self(), &policy, &param);
    cm->saved_priority = param.sched_priority;

    // 우선순위가 ceiling보다 높으면 에러
    if (param.sched_priority > cm->ceiling) {
        return EINVAL;  // Priority Ceiling 위반
    }

    // 우선순위를 ceiling으로 상승
    param.sched_priority = cm->ceiling;
    pthread_setschedparam(pthread_self(), policy, &param);

    // 실제 락 획득 (이제 선점되지 않음)
    return pthread_mutex_lock(&cm->mutex);
}

int ceiling_mutex_unlock(ceiling_mutex_t *cm) {
    // 락 해제
    int ret = pthread_mutex_unlock(&cm->mutex);

    // 원래 우선순위 복원
    struct sched_param param;
    int policy;
    pthread_getschedparam(pthread_self(), &policy, &param);
    param.sched_priority = cm->saved_priority;
    pthread_setschedparam(pthread_self(), policy, &param);

    return ret;
}

/*
 * Priority Ceiling 장점:
 *   - 교착 방지 (single-lock case)
 *   - 구현 단순
 *   - 예측 가능한 동작
 *
 * 단점:
 *   - ceiling을 미리 알아야 함
 *   - 불필요한 우선순위 상승 발생
 */
```

### VxWorks 우선순위 역전 해결 (Mars Pathfinder)

```c
// VxWorks semMCreate 옵션

// 문제가 된 원래 코드
SEM_ID dataSem = semMCreate(SEM_Q_PRIORITY);

// 수정된 코드 (priority inheritance 활성화)
SEM_ID dataSem = semMCreate(SEM_Q_PRIORITY | SEM_INVERSION_SAFE);

/*
 * SEM_INVERSION_SAFE 옵션:
 *   - 세마포어 보유자가 높은 우선순위 태스크에 의해
 *     블록되면 자동으로 우선순위 상승
 *   - 세마포어 해제 시 원래 우선순위 복원
 *
 * VxWorks 내부 구현:
 *   - 각 세마포어에 "소유자" 개념
 *   - 소유자의 pending priority list 관리
 *   - 블록 시 priority inheritance chain 구축
 */

// Mars Pathfinder 특정 상황:
// - 버스 관리 태스크 (Low): dataSem 보유
// - 통신 태스크 (Medium): 버스 관리 선점
// - 데이터 수집 태스크 (High): dataSem 대기 → 블록
//
// 해결 후:
// - High가 dataSem 대기 시 Low의 우선순위 → High로 상승
// - Medium이 Low를 선점 불가
// - Low가 빠르게 완료 → High 실행
// - Watchdog timeout 발생 안 함
```

## Detection and Analysis

### 1. Timeline Analysis

```c
#include <time.h>
#include <stdio.h>

typedef struct {
    pthread_t thread_id;
    int priority;
    struct timespec lock_time;
    struct timespec block_time;
    const char* resource_name;
} LockEvent;

LockEvent events[1000];
int num_events = 0;

void analyze_priority_inversion() {
    for (int i = 0; i < num_events - 1; i++) {
        // Find cases where high priority blocks on low priority
        if (events[i].priority > events[i+1].priority &&
            events[i].block_time.tv_sec > 0) {

            // Calculate blocking time
            long block_duration =
                (events[i+1].lock_time.tv_sec - events[i].block_time.tv_sec);

            printf("Priority Inversion Detected:\n");
            printf("  High-priority thread %lu (priority %d)\n",
                   events[i].thread_id, events[i].priority);
            printf("  Blocked for %ld seconds\n", block_duration);
            printf("  On resource: %s\n", events[i].resource_name);
        }
    }
}
```

### 2. Runtime Monitoring

```c
void monitor_mutex_operations() {
    // Hook into mutex lock/unlock
    // Track:
    // - Who holds lock
    // - Who is waiting
    // - Priorities of all involved threads

    if (waiting_priority > holder_priority) {
        printf("WARNING: Priority inversion detected!\n");
        printf("  Holder: priority %d\n", holder_priority);
        printf("  Waiter: priority %d\n", waiting_priority);

        // Check if holder can run
        for (each runnable thread) {
            if (thread_priority > holder_priority &&
                thread_priority < waiting_priority) {
                printf("  UNBOUNDED INVERSION: Medium priority thread %lu running\n",
                       thread_id);
            }
        }
    }
}
```

## Prevention Strategies

### Strategy 1: Avoid Shared Resources in RT Code

```c
// GOOD: High-priority threads don't share resources
void* realtime_thread(void* arg) {
    // Use lock-free algorithms
    atomic_int* shared_data = get_shared_data();
    atomic_store(shared_data, new_value);

    // No locks → No priority inversion!
}
```

### Strategy 2: Use Priority Inheritance

```c
// Enable for all real-time mutexes
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### Strategy 3: Minimize Critical Sections

```c
// BAD: Long critical section
pthread_mutex_lock(&mutex);
complex_computation();      // Can be preempted!
access_shared_data();
more_computation();
pthread_mutex_unlock(&mutex);

// GOOD: Minimal critical section
complex_computation();      // Do outside lock
pthread_mutex_lock(&mutex);
access_shared_data();       // Only shared access in lock
pthread_mutex_unlock(&mutex);
more_computation();
```

### Strategy 4: Use Disabling Interrupts (Embedded Systems)

```c
// For very short critical sections in embedded systems
void critical_operation() {
    disable_interrupts();
    // Very short operation
    access_hardware_register();
    enable_interrupts();
}

// Note: Only for very short sections!
// Can't be preempted → No priority inversion
```

## Testing for Priority Inversion

### Test Case Template

```c
#include <pthread.h>
#include <assert.h>
#include <time.h>

void test_priority_inversion() {
    pthread_t low, medium, high;
    struct timespec start, end;

    // Create threads with priorities: L=1, M=50, H=99
    // Low locks resource
    // Medium does work (no resource)
    // High waits for resource

    clock_gettime(CLOCK_MONOTONIC, &start);

    pthread_create(&low, NULL, low_priority, NULL);
    usleep(100000);  // Let low acquire lock

    pthread_create(&medium, NULL, medium_priority, NULL);
    usleep(100000);  // Let medium preempt

    pthread_create(&high, NULL, high_priority, NULL);

    pthread_join(high, NULL);
    clock_gettime(CLOCK_MONOTONIC, &end);

    long duration = (end.tv_sec - start.tv_sec) * 1000000000L +
                   (end.tv_nsec - start.tv_nsec);

    // Without priority inheritance: duration >> expected
    // With priority inheritance: duration ~= expected

    printf("High-priority thread completed in %ld ns\n", duration);

    pthread_cancel(low);
    pthread_cancel(medium);
}
```

## Best Practices

### For Real-Time Systems:

1. **Always use priority inheritance** for mutexes in RT code
2. **Minimize critical sections** as much as possible
3. **Avoid blocking** in high-priority threads if possible
4. **Use lock-free algorithms** when appropriate
5. **Test thoroughly** under worst-case scenarios
6. **Monitor for inversions** in production systems

### For General Systems:

1. **Document priority assumptions** clearly
2. **Use appropriate protocols** (inheritance or ceiling)
3. **Review critical paths** in code
4. **Profile and measure** actual behavior
5. **Consider alternatives** to priority-based scheduling

## Summary

**Priority Inversion** occurs when:
- High-priority thread waits for low-priority thread
- Medium-priority thread prevents low-priority from running
- High-priority effectively has lower priority than medium

**Famous Example**: Mars Pathfinder (1997)

**Solutions**:
1. **Priority Inheritance**: Boost low-priority when high-priority waits
2. **Priority Ceiling**: Always run at highest possible priority
3. **Avoid Sharing**: Use lock-free data structures
4. **Minimize Locks**: Reduce critical section duration

**Critical for**:
- Real-time systems
- Safety-critical applications
- Embedded systems
- Any priority-scheduled system

## Further Reading

- "What Really Happened on Mars?" - Glenn Reeves (JPL)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)
- "Real-Time Systems" - Jane W. S. Liu
- VxWorks documentation on priority inversion

## Conclusion

Priority inversion is a critical problem in real-time systems that can cause:
- Missed deadlines
- System instability
- Safety violations
- Mission failures (literally, as in Mars!)

Understanding and preventing priority inversion is essential for any system where timing guarantees matter.

---

**Congratulations!** You've completed the concurrency problems section. You now understand the five major concurrency problems and how to detect, prevent, and solve them. Continue to **04-synchronization-primitives/** to learn the tools for building correct concurrent programs.
