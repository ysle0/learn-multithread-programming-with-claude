# 우선순위 역전 (Priority Inversion)

## 우선순위 역전이란 무엇인가?

**우선순위 역전(Priority Inversion)**은 높은 우선순위 스레드가 낮은 우선순위 스레드에 의해 간접적으로 차단되어 우선순위 기반 스케줄링 보장을 위반하는 상황입니다. 이는 낮은 우선순위 스레드가 높은 우선순위 스레드가 필요로 하는 자원을 보유하고 있는 동안 중간 우선순위 스레드가 낮은 우선순위 스레드를 선점하여 자원을 해제하지 못하게 할 때 발생합니다.

### 간단한 용어로 설명한 문제

```
High Priority Thread:    "I need that resource NOW!"
                         ↓ (blocked)
Low Priority Thread:     "I have it, but I'm not running..."
                         ↓ (preempted)
Medium Priority Thread:  "I'm running instead!"
                         ↓ (prevents progress)

Result: High priority waits for medium priority (WRONG!)
```

## 시각적 표현

### 우선순위 역전 시나리오

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

### 자원 의존성 그래프

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

## 마스 패스파인더 사건 (1997년)

### 배경

마스 패스파인더는 1997년 7월 4일 화성에 착륙했습니다. 착륙 직후 우주선은 시스템 리셋을 경험하기 시작하여 데이터 손실과 임무 지연을 야기했습니다.

### 문제

```
Thread Priorities:
  High:   ASI/MET Bus Management Task (critical communication)
  Medium: Communications Task
  Low:    Meteorological Data Task

Shared Resource: Information Bus (protected by mutex)
```

### 무슨 일이 일어났는가

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

### 수정 방법

NASA 엔지니어들은 (VxWorks 전문가의 도움으로) VxWorks 뮤텍스 구현에서 **우선순위 상속**을 활성화했습니다.

```c
// Before: Normal mutex (no priority inheritance)
semaphore = semMCreate(SEM_Q_PRIORITY);

// After: Mutex with priority inheritance
semaphore = semMCreate(SEM_Q_PRIORITY | SEM_INVERSION_SAFE);
```

**어떻게 도움이 되었는가:**
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

## 우선순위 역전의 유형

### 1. 제한된 우선순위 역전

지속 시간이 낮은 우선순위 스레드의 임계 영역에 의해 제한됩니다.

```
Max delay = Length of L's critical section

Timeline:
  H: ───[BLOCKED]──── (bounded wait)
  L: ──[CRITICAL]──── (finishes quickly)
```

### 2. 무제한 우선순위 역전

지속 시간이 중간 우선순위 스레드에 의해 연장됩니다.

```
Max delay = Unknown (depends on M's execution time)

Timeline:
  H: ───[BLOCKED]──────────────────── (unbounded wait!)
  M: ───────[RUNNING]─────────────────
  L: ──[CRITICAL]──[PREEMPTED]────────
```

## 코드 예제

### 예제 1: 우선순위 역전 시연

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

### 예제 2: 우선순위 상속 해결책

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

### 예제 3: 우선순위 상한 프로토콜

우선순위 상속의 대안입니다.

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

## 해결책 비교

### 우선순위 상속

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

### 우선순위 상한

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

### 비교 표

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

## 실제 사례

### 예제 1: 실시간 제어 시스템

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

### 예제 2: 산업용 로봇 제어

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

### 예제 3: 의료 기기

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

## 탐지 및 분석

### 1. 타임라인 분석

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

### 2. 런타임 모니터링

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

## 예방 전략

### 전략 1: RT 코드에서 공유 자원 회피

```c
// GOOD: High-priority threads don't share resources
void* realtime_thread(void* arg) {
    // Use lock-free algorithms
    atomic_int* shared_data = get_shared_data();
    atomic_store(shared_data, new_value);

    // No locks → No priority inversion!
}
```

### 전략 2: 우선순위 상속 사용

```c
// Enable for all real-time mutexes
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### 전략 3: 임계 영역 최소화

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

### 전략 4: 인터럽트 비활성화 사용 (임베디드 시스템)

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

## 우선순위 역전 테스트

### 테스트 케이스 템플릿

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

## 모범 사례

### 실시간 시스템의 경우:

1. **항상 우선순위 상속 사용** RT 코드의 뮤텍스에 대해
2. **임계 영역을 최소화** 가능한 한 많이
3. **높은 우선순위 스레드에서 차단 회피** 가능한 경우
4. **적절한 경우 락 프리 알고리즘 사용**
5. **최악의 경우 시나리오에서 철저히 테스트**
6. **프로덕션 시스템에서 역전 모니터링**

### 일반 시스템의 경우:

1. **우선순위 가정을 명확하게 문서화**
2. **적절한 프로토콜 사용** (상속 또는 상한)
3. **코드의 임계 경로 검토**
4. **실제 동작 프로파일링 및 측정**
5. **우선순위 기반 스케줄링의 대안 고려**

## 요약

**우선순위 역전**은 다음과 같을 때 발생합니다:
- 높은 우선순위 스레드가 낮은 우선순위 스레드를 기다림
- 중간 우선순위 스레드가 낮은 우선순위가 실행되는 것을 방지
- 높은 우선순위가 사실상 중간보다 낮은 우선순위를 가짐

**유명한 예**: 마스 패스파인더 (1997년)

**해결책**:
1. **우선순위 상속**: 높은 우선순위가 대기할 때 낮은 우선순위를 부스트
2. **우선순위 상한**: 항상 가능한 최고 우선순위로 실행
3. **공유 회피**: 락 프리 데이터 구조 사용
4. **락 최소화**: 임계 영역 지속 시간 단축

**중요 대상**:
- 실시간 시스템
- 안전 중요 애플리케이션
- 임베디드 시스템
- 우선순위 스케줄링 시스템

## 추가 자료

- "What Really Happened on Mars?" - Glenn Reeves (JPL)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)
- "Real-Time Systems" - Jane W. S. Liu
- VxWorks documentation on priority inversion

## 결론

우선순위 역전은 다음을 야기할 수 있는 실시간 시스템의 중요한 문제입니다:
- 마감 시간 놓침
- 시스템 불안정성
- 안전 위반
- 임무 실패 (말 그대로 화성에서처럼!)

타이밍 보장이 중요한 모든 시스템에서 우선순위 역전을 이해하고 예방하는 것은 필수적입니다.

---

**축하합니다!** 동시성 문제 섹션을 완료했습니다. 이제 다섯 가지 주요 동시성 문제와 이를 탐지, 예방 및 해결하는 방법을 이해했습니다. **04-synchronization-primitives/**로 계속하여 올바른 동시 프로그램을 구축하기 위한 도구를 배우십시오.
