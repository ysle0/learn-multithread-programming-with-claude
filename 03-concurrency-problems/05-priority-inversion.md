# Priority Inversion

## Priority Inversion이란?

**Priority Inversion**은 높은 우선순위의 thread가 낮은 우선순위의 thread에 의해 간접적으로 차단되어, 우선순위 기반 스케줄링 보장을 위반하는 상황입니다. 이는 낮은 우선순위의 thread가 높은 우선순위의 thread가 필요로 하는 자원을 보유하고 있는 상태에서, 중간 우선순위의 thread가 낮은 우선순위의 thread를 선점하여 자원 해제를 방해할 때 발생합니다.

### 간단하게 설명하는 문제

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

### Priority Inversion 시나리오

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

## 화성 패스파인더 사건 (1997)

### 배경

화성 패스파인더는 1997년 7월 4일 화성에 착륙했습니다. 착륙 직후, 우주선은 시스템 리셋이 반복적으로 발생하여 데이터 손실과 임무 지연을 초래했습니다.

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

### 해결 방법

NASA 엔지니어들은 VxWorks 전문가의 도움을 받아 VxWorks mutex 구현에서 **priority inheritance**를 활성화했습니다.

```c
// 수정 전: 일반 mutex (priority inheritance 없음)
semaphore = semMCreate(SEM_Q_PRIORITY);

// 수정 후: priority inheritance가 있는 mutex
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

## Priority Inversion의 유형

### 1. 제한된 Priority Inversion (Bounded)

지속 시간이 낮은 우선순위 thread의 임계 영역에 의해 제한됩니다.

```
Max delay = Length of L's critical section

Timeline:
  H: ───[BLOCKED]──── (bounded wait)
  L: ──[CRITICAL]──── (finishes quickly)
```

### 2. 비제한 Priority Inversion (Unbounded)

지속 시간이 중간 우선순위 thread에 의해 연장됩니다.

```
Max delay = Unknown (depends on M's execution time)

Timeline:
  H: ───[BLOCKED]──────────────────── (unbounded wait!)
  M: ───────[RUNNING]─────────────────
  L: ──[CRITICAL]──[PREEMPTED]────────
```

## 코드 예제

### 예제 1: Priority Inversion 시연

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
    set_thread_priority(1);  // 가장 낮은 우선순위
    printf("[L] Starting\n");

    pthread_mutex_lock(&resource);
    printf("[L] Acquired resource\n");

    // 자원을 사용한 작업 시뮬레이션
    printf("[L] Working with resource (5 seconds)...\n");
    sleep(5);

    printf("[L] Releasing resource\n");
    pthread_mutex_unlock(&resource);

    return NULL;
}

void* medium_priority_thread(void* arg) {
    set_thread_priority(50);  // 중간 우선순위
    sleep(1);  // 낮은 우선순위 thread가 먼저 lock을 획득하도록 대기

    printf("[M] Starting - will preempt low priority!\n");

    // 연산 집약적 작업 (공유 자원 사용 안 함)
    printf("[M] Doing work (prevents low priority from finishing)...\n");
    for (volatile long i = 0; i < 1000000000L; i++);

    printf("[M] Finished\n");
    return NULL;
}

void* high_priority_thread(void* arg) {
    set_thread_priority(99);  // 가장 높은 우선순위
    sleep(2);  // 낮은 우선순위가 lock을 획득하고, 중간 우선순위가 선점하도록 대기

    printf("[H] Starting - NEED RESOURCE!\n");

    pthread_mutex_lock(&resource);
    printf("[H] Finally acquired resource (DELAYED by medium!)\n");

    // 중요 작업
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

// 예상 출력은 H가 M이 끝나기를 기다리는 것을 보여줍니다.
// M이 자원을 사용하지 않음에도 불구하고!
```

### 예제 2: Priority Inheritance 해결책

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

    // Priority inheritance 프로토콜 활성화
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

    // 획득 시도
    if (pthread_mutex_trylock(&pim->mutex) == 0) {
        // 즉시 획득 성공
        pim->owner = self;
        pim->owner_original_priority = my_param.sched_priority;
        return;
    }

    // 다른 thread가 소유 중 - 소유자의 우선순위 상승
    if (pim->owner) {
        struct sched_param owner_param;
        int owner_policy;
        pthread_getschedparam(pim->owner, &owner_policy, &owner_param);

        if (my_param.sched_priority > owner_param.sched_priority) {
            // 소유자의 우선순위를 현재 thread 수준으로 상승
            owner_param.sched_priority = my_param.sched_priority;
            pthread_setschedparam(pim->owner, owner_policy, &owner_param);

            printf("Priority inheritance: Boosted owner to priority %d\n",
                   my_param.sched_priority);
        }
    }

    // lock 대기
    pthread_mutex_lock(&pim->mutex);
    pim->owner = self;
    pim->owner_original_priority = my_param.sched_priority;
}

void pi_mutex_unlock(PriorityInheritanceMutex* pim) {
    pthread_t self = pthread_self();

    if (pim->owner == self) {
        // 우선순위가 상승되었다면 원래 우선순위 복원
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

// 이 구현은 더 높은 우선순위의 thread가 대기할 때
// lock 보유자의 우선순위를 자동으로 상승시킵니다
```

### 예제 3: Priority Ceiling 프로토콜

Priority inheritance의 대안입니다.

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

    // Priority ceiling 프로토콜 설정
    pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_PROTECT);
    pthread_mutexattr_setprioceiling(&attr, ceiling);

    pthread_mutex_init(&pcm->mutex, &attr);
    pthread_mutexattr_destroy(&attr);

    pcm->ceiling_priority = ceiling;
}

void pc_mutex_lock(PriorityCeilingMutex* pcm) {
    // lock을 보유하는 동안 thread는 자동으로 ceiling 우선순위로 실행
    pthread_mutex_lock(&pcm->mutex);

    printf("Lock acquired - running at ceiling priority %d\n",
           pcm->ceiling_priority);
}

void pc_mutex_unlock(PriorityCeilingMutex* pcm) {
    printf("Lock released - priority restored\n");
    pthread_mutex_unlock(&pcm->mutex);
}

// Priority ceiling 사용 시:
// - Lock 보유자는 항상 ceiling 우선순위로 실행
// - 중간 우선순위 thread가 선점 불가
// - Priority inversion을 완전히 방지!
```

## 해결책 비교

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
// 항공기 비행 제어 시스템
#define PRIORITY_CRITICAL   99  // 비행 제어
#define PRIORITY_HIGH       80  // 항법
#define PRIORITY_MEDIUM     50  // 통신
#define PRIORITY_LOW        20  // 로깅

pthread_mutex_t sensor_data_mutex;

void* critical_flight_control(void* arg) {
    set_priority(PRIORITY_CRITICAL);

    while (1) {
        // 반드시 10ms마다 실행되어야 함
        pthread_mutex_lock(&sensor_data_mutex);
        read_sensors();
        update_flight_controls();
        pthread_mutex_unlock(&sensor_data_mutex);

        usleep(10000);  // 10ms 주기
    }
}

void* medium_comms_task(void* arg) {
    set_priority(PRIORITY_MEDIUM);

    while (1) {
        // 중요 태스크를 지연시키면 안 됨!
        send_telemetry();
        sleep(1);
    }
}

void* low_logging_task(void* arg) {
    set_priority(PRIORITY_LOW);

    while (1) {
        pthread_mutex_lock(&sensor_data_mutex);
        log_sensor_data();  // 이것이 선점되면...
        pthread_mutex_unlock(&sensor_data_mutex);

        sleep(1);
    }
}

// Priority inheritance 없이:
// - 로깅이 sensor_data_mutex를 lock
// - 통신 태스크가 로깅을 선점
// - 비행 제어가 데드라인을 놓침!
// 결과: 치명적

// Priority inheritance 사용 시:
// - lock을 보유하는 동안 로깅이 CRITICAL로 상승
// - 통신이 선점 불가
// - 비행 제어가 데드라인 충족
// 결과: 안전
```

### 예제 2: 산업용 로봇 제어

```c
// 다중 제어 루프를 가진 로봇
typedef struct {
    PriorityInheritanceMutex position_mutex;
    double x, y, z;
} RobotPosition;

RobotPosition robot_pos;

void* servo_control(void* arg) {
    // 1kHz로 실행 - 반드시 빨라야 함
    set_priority(PRIORITY_REALTIME);

    while (1) {
        pi_mutex_lock(&robot_pos.position_mutex);

        // 서보 위치 업데이트
        update_servos(robot_pos.x, robot_pos.y, robot_pos.z);

        pi_mutex_unlock(&robot_pos.position_mutex);

        usleep(1000);  // 1ms 주기
    }
}

void* path_planning(void* arg) {
    // 중간 우선순위
    set_priority(PRIORITY_NORMAL);

    while (1) {
        // 다음 위치 계산
        calculate_path();
        usleep(10000);  // 10ms 주기
    }
}

void* ui_update(void* arg) {
    // 낮은 우선순위
    set_priority(PRIORITY_LOW);

    while (1) {
        pi_mutex_lock(&robot_pos.position_mutex);

        // 디스플레이 업데이트
        display_position(robot_pos.x, robot_pos.y, robot_pos.z);

        pi_mutex_unlock(&robot_pos.position_mutex);

        usleep(100000);  // 100ms 주기
    }
}

// Priority inheritance가 보장하는 것:
// - UI가 서보 제어를 절대 지연시키지 않음
// - 시스템이 응답성을 유지
// - 실시간 데드라인 충족
```

### 예제 3: 의료 기기

```c
// 인슐린 펌프 컨트롤러
#define PRIORITY_SAFETY      99  // 안전 모니터링
#define PRIORITY_DELIVERY    80  // 인슐린 전달
#define PRIORITY_UI          20  // 사용자 인터페이스

pthread_mutex_t dose_calculation_mutex;

void* safety_monitor(void* arg) {
    set_priority(PRIORITY_SAFETY);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);

        // 안전하지 않은 조건 확인
        if (blood_glucose_too_low() || pump_malfunction()) {
            emergency_stop();
        }

        pthread_mutex_unlock(&dose_calculation_mutex);

        usleep(100000);  // 100ms마다 확인
    }
}

void* insulin_delivery(void* arg) {
    set_priority(PRIORITY_DELIVERY);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);
        calculate_and_deliver_dose();
        pthread_mutex_unlock(&dose_calculation_mutex);

        sleep(5);  // 5분마다
    }
}

void* user_interface(void* arg) {
    set_priority(PRIORITY_UI);

    while (1) {
        pthread_mutex_lock(&dose_calculation_mutex);
        update_display();
        pthread_mutex_unlock(&dose_calculation_mutex);

        usleep(500000);  // 500ms마다 업데이트
    }
}

// 중요: UI가 안전 모니터링을 지연시키면 안 됨
// Priority inheritance는 환자 안전에 필수적
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
        // 높은 우선순위가 낮은 우선순위에 의해 차단되는 경우 탐색
        if (events[i].priority > events[i+1].priority &&
            events[i].block_time.tv_sec > 0) {

            // 차단 시간 계산
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
    // mutex lock/unlock에 대한 hook
    // 추적 항목:
    // - lock을 보유하고 있는 thread
    // - 대기 중인 thread
    // - 관련된 모든 thread의 우선순위

    if (waiting_priority > holder_priority) {
        printf("WARNING: Priority inversion detected!\n");
        printf("  Holder: priority %d\n", holder_priority);
        printf("  Waiter: priority %d\n", waiting_priority);

        // 보유자가 실행 가능한지 확인
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

### 전략 1: 실시간 코드에서 공유 자원 회피

```c
// 좋은 예: 높은 우선순위 thread가 자원을 공유하지 않음
void* realtime_thread(void* arg) {
    // Lock-free 알고리즘 사용
    atomic_int* shared_data = get_shared_data();
    atomic_store(shared_data, new_value);

    // Lock 없음 → Priority inversion 없음!
}
```

### 전략 2: Priority Inheritance 사용

```c
// 모든 실시간 mutex에 대해 활성화
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### 전략 3: 임계 영역 최소화

```c
// 나쁜 예: 긴 임계 영역
pthread_mutex_lock(&mutex);
complex_computation();      // 선점될 수 있음!
access_shared_data();
more_computation();
pthread_mutex_unlock(&mutex);

// 좋은 예: 최소한의 임계 영역
complex_computation();      // lock 외부에서 수행
pthread_mutex_lock(&mutex);
access_shared_data();       // 공유 접근만 lock 내부에서
pthread_mutex_unlock(&mutex);
more_computation();
```

### 전략 4: 인터럽트 비활성화 사용 (임베디드 시스템)

```c
// 임베디드 시스템의 매우 짧은 임계 영역에 사용
void critical_operation() {
    disable_interrupts();
    // 매우 짧은 작업
    access_hardware_register();
    enable_interrupts();
}

// 주의: 매우 짧은 구간에만 사용!
// 선점 불가 → Priority inversion 없음
```

## Priority Inversion 테스트

### 테스트 케이스 템플릿

```c
#include <pthread.h>
#include <assert.h>
#include <time.h>

void test_priority_inversion() {
    pthread_t low, medium, high;
    struct timespec start, end;

    // 우선순위가 L=1, M=50, H=99인 thread 생성
    // Low가 자원을 lock
    // Medium이 작업 수행 (자원 미사용)
    // High가 자원 대기

    clock_gettime(CLOCK_MONOTONIC, &start);

    pthread_create(&low, NULL, low_priority, NULL);
    usleep(100000);  // Low가 lock을 획득하도록 대기

    pthread_create(&medium, NULL, medium_priority, NULL);
    usleep(100000);  // Medium이 선점하도록 대기

    pthread_create(&high, NULL, high_priority, NULL);

    pthread_join(high, NULL);
    clock_gettime(CLOCK_MONOTONIC, &end);

    long duration = (end.tv_sec - start.tv_sec) * 1000000000L +
                   (end.tv_nsec - start.tv_nsec);

    // Priority inheritance 없이: duration >> 예상값
    // Priority inheritance 사용 시: duration ~= 예상값

    printf("High-priority thread completed in %ld ns\n", duration);

    pthread_cancel(low);
    pthread_cancel(medium);
}
```

## 모범 사례

### 실시간 시스템의 경우:

1. **실시간 코드의 mutex에는 항상 priority inheritance를 사용**하세요
2. **임계 영역을 가능한 한 최소화**하세요
3. 가능하면 **높은 우선순위 thread에서 blocking을 회피**하세요
4. 적절한 경우 **lock-free 알고리즘을 사용**하세요
5. 최악의 시나리오에서 **철저히 테스트**하세요
6. 운영 시스템에서 **inversion을 모니터링**하세요

### 일반 시스템의 경우:

1. **우선순위 가정을 명확히 문서화**하세요
2. **적절한 프로토콜** (inheritance 또는 ceiling)을 사용하세요
3. 코드의 **중요 경로를 검토**하세요
4. 실제 동작을 **프로파일링하고 측정**하세요
5. 우선순위 기반 스케줄링의 **대안을 고려**하세요

## 요약

**Priority Inversion**은 다음과 같은 경우에 발생합니다:
- 높은 우선순위 thread가 낮은 우선순위 thread를 기다림
- 중간 우선순위 thread가 낮은 우선순위 thread의 실행을 방해
- 높은 우선순위가 사실상 중간 우선순위보다 낮은 우선순위를 가짐

**유명한 사례**: 화성 패스파인더 (1997)

**해결책**:
1. **Priority Inheritance**: 높은 우선순위가 대기할 때 낮은 우선순위를 상승
2. **Priority Ceiling**: 항상 가능한 가장 높은 우선순위로 실행
3. **공유 회피**: Lock-free 자료 구조 사용
4. **Lock 최소화**: 임계 영역 지속 시간 단축

**다음과 같은 분야에서 중요**:
- 실시간 시스템
- 안전 필수 애플리케이션
- 임베디드 시스템
- 모든 우선순위 스케줄링 기반 시스템

## 추가 읽기

- "What Really Happened on Mars?" - Glenn Reeves (JPL)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)
- "Real-Time Systems" - Jane W. S. Liu
- VxWorks documentation on priority inversion

## 결론

Priority inversion은 실시간 시스템에서 다음을 초래할 수 있는 치명적인 문제입니다:
- 데드라인 미충족
- 시스템 불안정
- 안전 위반
- 임무 실패 (화성에서처럼, 말 그대로!)

타이밍 보장이 중요한 모든 시스템에서 priority inversion을 이해하고 예방하는 것은 필수적입니다.

---

**축하합니다!** 동시성 문제 섹션을 완료했습니다. 이제 다섯 가지 주요 동시성 문제와 이를 탐지하고, 예방하고, 해결하는 방법을 이해하게 되었습니다. 올바른 동시성 프로그램을 구축하기 위한 도구를 배우려면 **04-synchronization-primitives/**로 계속 진행하세요.
