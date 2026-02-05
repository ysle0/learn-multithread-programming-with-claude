# Deadlock

## Deadlock이란?

**Deadlock**은 두 개 이상의 thread가 서로가 보유한 자원을 기다리면서 영원히 차단되는 상황입니다. 어떤 thread도 진행할 수 없는 순환 대기 조건으로, 영구적인 교착 상태를 초래합니다.

### 공식적 정의

Deadlock은 다음 네 가지 조건(Coffman 조건)이 동시에 모두 충족될 때 발생합니다:

1. **상호 배제(Mutual Exclusion)**: 자원을 공유할 수 없음
2. **점유 및 대기(Hold and Wait)**: Thread가 자원을 보유한 채 다른 자원을 기다림
3. **비선점(No Preemption)**: 자원을 강제로 빼앗을 수 없음
4. **순환 대기(Circular Wait)**: Thread 간 자원 대기의 순환 고리가 존재

## 시각적 표현

### 간단한 두 Thread Deadlock

```
Thread 1                          Thread 2
--------                          --------
Lock(Mutex A) ✓                   Lock(Mutex B) ✓
    |                                 |
    | Waiting for Mutex B             | Waiting for Mutex A
    |        ↓                         |        ↓
    └────────┼─────────────────────────┘
             ↓
          DEADLOCK!

┌─────────┐              ┌─────────┐
│Thread 1 │─── waits ──→ │ Mutex B │
│         │              │         │
│ holds   │              │ held by │
│         │              │         │
│ Mutex A │←─── waits ── │Thread 2 │
└─────────┘              └─────────┘
        Circular dependency = Deadlock
```

### 자원 할당 그래프

```
         P1 (Thread 1)
          ↓ owns
    ┌──→ R1 (Resource/Lock 1)
    │     ↑ needs
    │    P2 (Thread 2)
    │     ↓ owns
    └─── R2 (Resource/Lock 2)
          ↑ needs
          P1

Cycle detected → Deadlock exists!
```

## 네 가지 필요 조건 (Coffman 조건)

### 1. 상호 배제(Mutual Exclusion)

자원을 공유할 수 없으며, 한 번에 하나의 thread만 자원을 사용할 수 있습니다.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// 오직 하나의 thread만 이 mutex를 보유할 수 있음
pthread_mutex_lock(&mutex);
// 임계 영역 - 배타적 접근
pthread_mutex_unlock(&mutex);
```

### 2. 점유 및 대기(Hold and Wait)

최소 하나의 자원을 보유한 thread가 다른 thread가 보유한 추가 자원의 획득을 기다리고 있는 상태입니다.

```c
// Thread 1이 A를 보유하고 B를 기다림
pthread_mutex_lock(&mutex_a);  // A를 보유 중
// ... 작업 수행 ...
pthread_mutex_lock(&mutex_b);  // B를 기다림
```

### 3. 비선점(No Preemption)

자원을 thread로부터 강제로 제거할 수 없으며, 자발적으로 해제해야 합니다.

```c
// 한 번 잠그면 강제로 빼앗을 수 없음
pthread_mutex_lock(&mutex);
// ... 더 높은 우선순위의 thread가 필요하더라도 ...
pthread_mutex_unlock(&mutex);  // 자발적으로 해제해야 함
```

### 4. 순환 대기(Circular Wait)

각 thread가 체인의 다음 thread가 보유한 자원을 기다리는 순환 고리가 존재합니다.

```
T1 waits for resource held by T2
T2 waits for resource held by T3
T3 waits for resource held by T1
    ↓
Circular dependency!
```

## 고전적 예제: 식사하는 철학자 문제

### 문제 설명

다섯 명의 철학자가 다섯 개의 포크가 놓인 원탁에 앉아 있습니다. 각 철학자는 식사를 하려면 두 개의 포크가 필요하지만, 각 철학자 사이에는 포크가 하나밖에 없습니다.

```
           Fork 0
              │
        P0 ←──┘──→ P1
        ↑           ↓
    Fork 4       Fork 1
        ↑           ↓
        P4 ←─────→ P2
              ↓
        Fork 3   Fork 2
              P3
```

### Deadlock 구현

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

#define NUM_PHILOSOPHERS 5

pthread_mutex_t forks[NUM_PHILOSOPHERS];

void* philosopher(void* arg) {
    int id = *(int*)arg;
    int left_fork = id;
    int right_fork = (id + 1) % NUM_PHILOSOPHERS;

    while (1) {
        // 생각하기
        printf("Philosopher %d is thinking\n", id);
        sleep(1);

        // 왼쪽 포크 집기
        printf("Philosopher %d picks up left fork %d\n", id, left_fork);
        pthread_mutex_lock(&forks[left_fork]);

        // 오른쪽 포크 집기 - 여기서 DEADLOCK이 발생할 수 있음!
        printf("Philosopher %d picks up right fork %d\n", id, right_fork);
        pthread_mutex_lock(&forks[right_fork]);

        // 식사하기
        printf("Philosopher %d is eating\n", id);
        sleep(2);

        // 포크 내려놓기
        pthread_mutex_unlock(&forks[right_fork]);
        pthread_mutex_unlock(&forks[left_fork]);
        printf("Philosopher %d finished eating\n", id);
    }

    return NULL;
}

int main() {
    pthread_t philosophers[NUM_PHILOSOPHERS];
    int ids[NUM_PHILOSOPHERS];

    // 포크 초기화
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_mutex_init(&forks[i], NULL);
    }

    // 철학자 생성
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        ids[i] = i;
        pthread_create(&philosophers[i], NULL, philosopher, &ids[i]);
    }

    // 영원히 대기 (deadlock 발생)
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_join(philosophers[i], NULL);
    }

    return 0;
}
```

**발생하는 상황:**
1. 모든 철학자가 동시에 왼쪽 포크를 집음
2. 모든 철학자가 오른쪽 포크를 집으려 시도
3. 모든 오른쪽 포크는 이미 이웃의 왼쪽 포크로 사용되고 있음
4. **DEADLOCK**: 모두가 영원히 기다림

## 예방 전략

네 가지 Coffman 조건 중 하나라도 제거하면 deadlock을 예방할 수 있습니다.

### 전략 1: 상호 배제 제거

자원을 공유 가능하게 만듭니다 (항상 가능하지는 않음).

```c
// 읽기 위주의 데이터에 read-write lock 사용
pthread_rwlock_t rwlock;

// 여러 reader가 동시에 보유 가능
pthread_rwlock_rdlock(&rwlock);  // 공유 접근
read_data();
pthread_rwlock_unlock(&rwlock);

// writer는 여전히 배타적 접근이 필요
pthread_rwlock_wrlock(&rwlock);  // 배타적 접근
write_data();
pthread_rwlock_unlock(&rwlock);
```

### 전략 2: 점유 및 대기 제거

모든 자원을 한꺼번에 획득하거나, 아예 획득하지 않습니다.

```c
// 해결책: 전부 아니면 전무 방식의 자원 획득
pthread_mutex_t global_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_b = PTHREAD_MUTEX_INITIALIZER;

void critical_section() {
    // 전역 lock을 사용하여 두 mutex를 원자적으로 획득
    pthread_mutex_lock(&global_lock);

    pthread_mutex_lock(&mutex_a);
    pthread_mutex_lock(&mutex_b);

    pthread_mutex_unlock(&global_lock);

    // 두 자원으로 작업 수행
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

**trylock을 사용한 더 나은 접근법:**
```c
#include <pthread.h>
#include <stdbool.h>
#include <time.h>

bool acquire_both(pthread_mutex_t* m1, pthread_mutex_t* m2) {
    pthread_mutex_lock(m1);

    if (pthread_mutex_trylock(m2) == 0) {
        return true;  // 두 lock 모두 획득
    }

    // 두 번째 lock을 얻지 못했으므로 첫 번째 해제
    pthread_mutex_unlock(m1);
    return false;  // 둘 다 획득 실패
}

void critical_section() {
    while (!acquire_both(&mutex_a, &mutex_b)) {
        // 잠시 대기 후 재시도
        struct timespec ts = {0, 100000};  // 100 마이크로초
        nanosleep(&ts, NULL);
    }

    // 두 자원으로 작업 수행
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

### 전략 3: 선점 허용

타임아웃을 사용하여 대기를 포기합니다.

```c
#include <pthread.h>
#include <time.h>
#include <errno.h>

void critical_section_with_timeout() {
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 1;  // 1초 타임아웃

    pthread_mutex_lock(&mutex_a);

    int result = pthread_mutex_timedlock(&mutex_b, &timeout);

    if (result == ETIMEDOUT) {
        // 획득할 수 없으므로 해제 후 재시도
        pthread_mutex_unlock(&mutex_a);
        printf("Timeout! Backing off...\n");
        return;
    }

    // 두 자원으로 작업 수행
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

### 전략 4: 순환 대기 제거

**Lock 순서화**: 항상 일관된 전역 순서로 lock을 획득합니다.

```c
// 해결책: 순서화된 lock 획득
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t mutex_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_b = PTHREAD_MUTEX_INITIALIZER;

void thread1_work() {
    // 항상 A를 B보다 먼저 잠금
    pthread_mutex_lock(&mutex_a);
    printf("Thread 1: Locked A\n");

    pthread_mutex_lock(&mutex_b);
    printf("Thread 1: Locked B\n");

    // 임계 영역
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}

void thread2_work() {
    // 항상 A를 B보다 먼저 잠금 (같은 순서!)
    pthread_mutex_lock(&mutex_a);
    printf("Thread 2: Locked A\n");

    pthread_mutex_lock(&mutex_b);
    printf("Thread 2: Locked B\n");

    // 임계 영역
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

**Lock 순서화를 적용한 식사하는 철학자:**
```c
void* philosopher_ordered(void* arg) {
    int id = *(int*)arg;
    int left_fork = id;
    int right_fork = (id + 1) % NUM_PHILOSOPHERS;

    // 해결책: 항상 번호가 낮은 포크를 먼저 집기
    int first_fork = (left_fork < right_fork) ? left_fork : right_fork;
    int second_fork = (left_fork < right_fork) ? right_fork : left_fork;

    while (1) {
        printf("Philosopher %d is thinking\n", id);
        sleep(1);

        // 순서대로 포크 집기
        pthread_mutex_lock(&forks[first_fork]);
        pthread_mutex_lock(&forks[second_fork]);

        // 식사하기
        printf("Philosopher %d is eating\n", id);
        sleep(2);

        // 포크 내려놓기
        pthread_mutex_unlock(&forks[second_fork]);
        pthread_mutex_unlock(&forks[first_fork]);
    }

    return NULL;
}
```

## Internal Mechanisms

### 커널 수준 교착 탐지

Linux 커널은 락 의존성 검증기(Lock Dependency Validator, lockdep)를 통해 잠재적 교착 상태를 탐지합니다.

#### Lockdep 구조

```c
// Linux kernel: include/linux/lockdep.h
struct lock_class {
    struct hlist_node       hash_entry;
    struct list_head        lock_entry;

    // 락 그래프에서의 의존성
    struct list_head        locks_after;   // 이 락 이후에 잡힌 락들
    struct list_head        locks_before;  // 이 락 이전에 잡힌 락들

    const char              *name;
    int                     name_version;

    unsigned long           usage_mask;    // 사용 패턴 (IRQ context 등)
    struct lock_class_key   *key;
};

// 락 체인: 락 획득 순서 추적
struct lock_chain {
    u64                     chain_key;     // 해시 키
    int                     depth;         // 체인 깊이
    int                     base;          // 체인 시작 인덱스
    struct hlist_node       entry;
};
```

#### 의존성 그래프 구축

```
Lockdep가 추적하는 락 획득 순서:

Thread 1: lock(A) → lock(B) → unlock(B) → unlock(A)
Thread 2: lock(B) → lock(C) → unlock(C) → unlock(B)

생성되는 의존성 그래프:
   A → B  (A를 잡은 후 B를 잡음)
   B → C  (B를 잡은 후 C를 잡음)

만약 Thread 3가: lock(C) → lock(A) 시도하면:
   C → A 추가 시도
   사이클 탐지: A → B → C → A
   WARNING 출력!
```

#### Lockdep 사이클 탐지 알고리즘

```c
// 간략화된 사이클 탐지 (DFS 기반)
static int check_deadlock(struct lock_class *source,
                         struct lock_class *target) {
    struct list_head *entry;
    struct lock_list *lock;

    // source에서 target으로 가는 경로가 있으면 교착 가능
    list_for_each(entry, &source->locks_after) {
        lock = list_entry(entry, struct lock_list, entry);

        if (lock->class == target) {
            // 직접 의존성: source → target
            // 역방향 의존성 추가 시 사이클!
            print_deadlock_warning(source, target);
            return 1;
        }

        // 재귀적으로 검사
        if (check_deadlock(lock->class, target))
            return 1;
    }

    return 0;
}

// 새로운 락 의존성 추가 시 호출
static int add_lock_dependency(struct lock_class *prev,
                               struct lock_class *next) {
    // 역방향 경로 존재 확인 (사이클 검사)
    if (check_deadlock(next, prev)) {
        // 교착 가능! prev → next 추가 시 사이클 형성
        return -EDEADLK;
    }

    // 안전: 의존성 추가
    list_add(&next->entry, &prev->locks_after);
    list_add(&prev->entry, &next->locks_before);

    return 0;
}
```

#### Lockdep 출력 예시

```
=============================================
[ INFO: possible circular locking dependency detected ]
5.10.0-rc1 #1 Not tainted
----------------------------------------------
test/1234 is trying to acquire lock:
ffff8881234abcde (&mutex_B){+.+.}-{3:3}, at: function_b+0x40/0x100

but task is already holding lock:
ffff8881234def01 (&mutex_A){+.+.}-{3:3}, at: function_a+0x30/0x80

which lock already depends on the new lock.

the existing dependency chain (in reverse order) is:

-> #1 (&mutex_A){+.+.}-{3:3}:
       lock_acquire+0xb0/0x200
       __mutex_lock+0x80/0x900
       mutex_lock+0x10/0x30
       function_x+0x20/0x60

-> #0 (&mutex_B){+.+.}-{3:3}:
       lock_acquire+0xb0/0x200
       __mutex_lock+0x80/0x900
       mutex_lock+0x10/0x30
       function_a+0x30/0x80

other info that might help us debug this:
 Possible unsafe locking scenario:
       CPU0                    CPU1
       ----                    ----
  lock(&mutex_A);
                               lock(&mutex_B);
                               lock(&mutex_A);
  lock(&mutex_B);

 *** DEADLOCK ***
```

### Wait-For Graph 구현

```c
// 사용자 공간에서의 Wait-For Graph 구현
typedef struct {
    int thread_id;
    int waiting_for_thread;  // -1 if not waiting
    pthread_mutex_t* held_locks[MAX_LOCKS];
    int num_held_locks;
} ThreadLockInfo;

ThreadLockInfo thread_info[MAX_THREADS];

// 교착 탐지 (사이클 탐지)
bool detect_deadlock_cycle() {
    // 방문 상태: 0=미방문, 1=방문중, 2=완료
    int visited[MAX_THREADS] = {0};

    for (int i = 0; i < MAX_THREADS; i++) {
        if (visited[i] == 0) {
            if (dfs_detect_cycle(i, visited)) {
                return true;  // 교착 발견!
            }
        }
    }
    return false;
}

bool dfs_detect_cycle(int thread_id, int* visited) {
    visited[thread_id] = 1;  // 방문 중

    int waiting_for = thread_info[thread_id].waiting_for_thread;
    if (waiting_for >= 0) {
        if (visited[waiting_for] == 1) {
            // 방문 중인 노드 재방문 = 사이클!
            print_deadlock_cycle(thread_id, waiting_for);
            return true;
        }
        if (visited[waiting_for] == 0) {
            if (dfs_detect_cycle(waiting_for, visited))
                return true;
        }
    }

    visited[thread_id] = 2;  // 완료
    return false;
}
```

### pthread_mutex 내부의 교착 탐지

```c
// glibc pthread_mutex의 에러 체킹 모드
// PTHREAD_MUTEX_ERRORCHECK 타입 사용 시

int pthread_mutex_lock(pthread_mutex_t *mutex) {
    int type = mutex->__data.__kind & PTHREAD_MUTEX_KIND_MASK;

    if (type == PTHREAD_MUTEX_ERRORCHECK_NP) {
        // 소유권 확인
        if (mutex->__data.__owner == pthread_self()) {
            // 같은 스레드가 이미 보유 중 = 잠재적 교착!
            return EDEADLK;
        }
    }

    // 실제 락 획득 시도
    int result = lll_lock(&mutex->__data.__lock, ...);

    if (result == 0) {
        mutex->__data.__owner = pthread_self();
    }

    return result;
}
```

### 타임아웃 기반 교착 회피

```c
// POSIX 타임아웃 락의 커널 구현
// linux/kernel/futex.c (간략화)

static int futex_lock_timeout(u32 __user *uaddr,
                             ktime_t *timeout) {
    struct futex_hash_bucket *hb;
    struct futex_q q;
    int ret;

    // 타임아웃 설정
    hrtimer_init_sleeper(&timeout_timer, current);
    hrtimer_start(&timeout_timer, *timeout, HRTIMER_MODE_ABS);

retry:
    // Fast path: 락 획득 시도
    if (futex_trylock(uaddr)) {
        hrtimer_cancel(&timeout_timer);
        return 0;
    }

    // Slow path: 대기
    futex_queue(&q, hb);

    // 타임아웃 또는 락 해제 대기
    set_current_state(TASK_INTERRUPTIBLE);

    if (timeout_timer.task == NULL) {
        // 타임아웃 발생!
        ret = -ETIMEDOUT;
        goto out;
    }

    schedule();

    if (signal_pending(current)) {
        ret = -EINTR;
        goto out;
    }

    goto retry;

out:
    hrtimer_cancel(&timeout_timer);
    return ret;
}
```

### 락 계층 구조 (Lock Hierarchy)

```c
// 락 순서를 강제하는 시스템

typedef struct {
    pthread_mutex_t mutex;
    int level;              // 계층 레벨 (낮을수록 먼저 획득)
    const char* name;
} HierarchicalMutex;

// 스레드별 현재 보유 중인 최고 레벨
__thread int current_max_level = -1;

int hierarchical_lock(HierarchicalMutex* hm) {
    // 락 순서 검증
    if (hm->level <= current_max_level) {
        fprintf(stderr, "Lock order violation! "
                "Trying to acquire level %d (%s) "
                "while holding level %d\n",
                hm->level, hm->name, current_max_level);
        // 옵션: abort() 또는 에러 반환
        abort();
    }

    int ret = pthread_mutex_lock(&hm->mutex);
    if (ret == 0) {
        current_max_level = hm->level;
    }
    return ret;
}

void hierarchical_unlock(HierarchicalMutex* hm) {
    pthread_mutex_unlock(&hm->mutex);
    // 레벨 복원 로직 (스택 사용 필요)
}

/*
사용 예시:
  HierarchicalMutex db_lock     = {.level = 100, .name = "db"};
  HierarchicalMutex table_lock  = {.level = 200, .name = "table"};
  HierarchicalMutex row_lock    = {.level = 300, .name = "row"};

  // 항상 db → table → row 순서로 획득
*/
```

## 탐지 전략

### 1. 자원 할당 그래프

자원과 thread의 그래프를 구축하여 순환을 탐지합니다.

```c
typedef struct {
    int thread_id;
    int* held_resources;
    int* waiting_for;
} ThreadInfo;

bool detect_cycle(ThreadInfo* threads, int num_threads) {
    // DFS를 사용하여 wait-for 그래프에서 순환 탐지
    // 순환이 발견되면 deadlock이 존재
    // 구현: 그래프 순회 알고리즘
}
```

### 2. Wait-For 그래프

자원 할당 그래프보다 간단하며, thread 의존성만 추적합니다.

```
Thread Dependencies:
T1 → T2  (T1 waits for T2)
T2 → T3  (T2 waits for T3)
T3 → T1  (T3 waits for T1)

Cycle: T1 → T2 → T3 → T1
Result: DEADLOCK DETECTED
```

### 3. 런타임 탐지 도구

```bash
# Helgrind (Valgrind) 사용
valgrind --tool=helgrind ./program

# 출력 예시:
# Thread #1: lock order "0x4C0D040 before 0x4C0D080" violated
# Thread #2: lock order "0x4C0D080 before 0x4C0D040" violated
# => Possible deadlock detected

# GDB를 사용하여 deadlock된 프로그램 검사
gdb -p <pid>
(gdb) info threads
(gdb) thread apply all bt  # 모든 thread의 백트레이스
```

### 4. 타임아웃 기반 탐지

```c
#include <pthread.h>
#include <time.h>
#include <stdio.h>

void detect_with_timeout() {
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 5;  // 5초 타임아웃

    int result = pthread_mutex_timedlock(&mutex, &timeout);

    if (result == ETIMEDOUT) {
        printf("DEADLOCK suspected: timeout after 5 seconds\n");
        // 교정 조치 수행
    }
}
```

## 복구 전략

### 1. Thread 종료

순환을 끊기 위해 하나 이상의 thread를 종료합니다.

```c
// Deadlock 탐지 후:
pthread_cancel(deadlocked_thread);
// 또는
pthread_kill(deadlocked_thread, SIGTERM);
```

### 2. 자원 선점

Thread가 자원을 강제로 해제하도록 합니다.

```c
// 실제로는 어려움 - 신중한 상태 관리가 필요
void force_release(Thread* victim) {
    // victim의 작업을 롤백
    // 자원 해제
    // victim 재시작
}
```

### 3. 롤백 및 재시작

체크포인트를 저장하고 deadlock 탐지 시 롤백합니다.

```c
typedef struct {
    void* saved_state;
    pthread_mutex_t* held_locks;
} Checkpoint;

void rollback_on_deadlock(Checkpoint* cp) {
    // 모든 lock 해제
    for (int i = 0; i < cp->num_locks; i++) {
        pthread_mutex_unlock(&cp->held_locks[i]);
    }

    // 상태 복원
    restore_state(cp->saved_state);

    // 작업 재시도
}
```

## 실제 사례

### 예제 1: 데이터베이스 Deadlock

```c
// Transaction 1:
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  // Lock row 1
// ... context switch ...
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  // Wait for row 2
COMMIT;

// Transaction 2:
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;   // Lock row 2
// ... context switch ...
UPDATE accounts SET balance = balance + 50 WHERE id = 1;   // Wait for row 1
COMMIT;

// DEADLOCK! T1이 T2를 기다리고, T2가 T1을 기다림
```

**해결책: 일관된 순서화**
```sql
-- 항상 ID 순서대로 계좌를 업데이트
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 낮은 ID 먼저
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 높은 ID 나중에
COMMIT;
```

### 예제 2: 파일 시스템 Deadlock

```c
// Thread 1: /a에서 /b로 파일 이동
lock_directory("/a");
lock_directory("/b");
move_file("/a/file.txt", "/b/file.txt");
unlock_directory("/b");
unlock_directory("/a");

// Thread 2: /b에서 /a로 파일 이동
lock_directory("/b");  // 반대 순서로 잠금!
lock_directory("/a");
move_file("/b/other.txt", "/a/other.txt");
unlock_directory("/a");
unlock_directory("/b");

// DEADLOCK 발생 가능!
```

**해결책: 디렉토리 경로를 알파벳 순서로 잠금**
```c
void move_file_safe(const char* from_dir, const char* to_dir,
                   const char* filename) {
    // lock 순서 결정
    const char* first = (strcmp(from_dir, to_dir) < 0) ? from_dir : to_dir;
    const char* second = (strcmp(from_dir, to_dir) < 0) ? to_dir : from_dir;

    lock_directory(first);
    lock_directory(second);

    // 이동 수행
    char from_path[256], to_path[256];
    snprintf(from_path, sizeof(from_path), "%s/%s", from_dir, filename);
    snprintf(to_path, sizeof(to_path), "%s/%s", to_dir, filename);
    rename(from_path, to_path);

    unlock_directory(second);
    unlock_directory(first);
}
```

### 예제 3: 네트워크 프로토콜 Deadlock

```c
// Node A가 B에게 전송, ACK 대기
send_to(node_b, data);
wait_for_ack_from(node_b);

// Node B가 A에게 전송, ACK 대기
send_to(node_a, data);
wait_for_ack_from(node_a);

// 양쪽 버퍼 모두 가득 참 → DEADLOCK!
```

**해결책: 타임아웃 및 재시도**
```c
bool send_with_timeout(Node* target, Data* data, int timeout_ms) {
    send_to(target, data);

    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += timeout_ms / 1000;

    if (wait_for_ack_timeout(target, &timeout) == ETIMEDOUT) {
        return false;  // 타임아웃, deadlock 없음
    }

    return true;
}
```

## 고급 패턴

### 계층적 잠금(Hierarchical Locking)

```c
// lock 계층 레벨 정의
#define LEVEL_DATABASE  100
#define LEVEL_TABLE     200
#define LEVEL_ROW       300

typedef struct {
    pthread_mutex_t mutex;
    int level;
} HierarchicalMutex;

// 증가하는 레벨 순서로만 lock 획득 가능
void hierarchical_lock(HierarchicalMutex* m, int current_level) {
    if (m->level <= current_level) {
        fprintf(stderr, "Lock ordering violation!\n");
        abort();
    }
    pthread_mutex_lock(&m->mutex);
}
```

### Try-Lock과 백오프

```c
#include <pthread.h>
#include <unistd.h>
#include <stdbool.h>

bool try_acquire_with_backoff(pthread_mutex_t* m1, pthread_mutex_t* m2) {
    int backoff = 1000;  // 1ms부터 시작

    for (int attempts = 0; attempts < 10; attempts++) {
        pthread_mutex_lock(m1);

        if (pthread_mutex_trylock(m2) == 0) {
            return true;  // 성공!
        }

        // 실패, 해제 후 백오프
        pthread_mutex_unlock(m1);
        usleep(backoff);
        backoff *= 2;  // 지수 백오프
    }

    return false;  // 10회 시도 후 포기
}
```

### Lock-Free 대안

```c
// Lock-free 구조로 deadlock을 완전히 회피
#include <stdatomic.h>

typedef struct Node {
    int value;
    struct Node* next;
} Node;

typedef struct {
    atomic_uintptr_t head;
} LockFreeStack;

void push(LockFreeStack* stack, int value) {
    Node* new_node = malloc(sizeof(Node));
    new_node->value = value;

    Node* old_head;
    do {
        old_head = (Node*)atomic_load(&stack->head);
        new_node->next = old_head;
    } while (!atomic_compare_exchange_weak(&stack->head,
                                          (uintptr_t*)&old_head,
                                          (uintptr_t)new_node));
}

// Lock이 없으므로 deadlock 발생 불가!
```

## 모범 사례 요약

### 해야 할 것:
- 일관된 lock 순서 사용
- 임계 영역 최소화
- trylock과 백오프 사용
- 타임아웃 구현
- Deadlock 탐지 도구로 테스트
- Lock 계층 문서화
- Lock-free 대안 고려

### 하지 말아야 할 것:
- Lock을 보유한 채 I/O 대기
- 서로 다른 순서로 lock 획득
- Lock을 보유한 채 알 수 없는 코드 호출
- 피할 수 있다면 여러 lock을 동시에 보유
- 무한정 차단

## 연습 문제

### 연습 문제 1: Deadlock 수정하기
```c
void transfer(Account* from, Account* to, int amount) {
    pthread_mutex_lock(&from->mutex);
    pthread_mutex_lock(&to->mutex);

    from->balance -= amount;
    to->balance += amount;

    pthread_mutex_unlock(&to->mutex);
    pthread_mutex_unlock(&from->mutex);
}

// 이 코드는 deadlock이 발생할 수 있습니다! 수정하세요.
```

### 연습 문제 2: 안전한 식사하는 철학자 구현
비대칭 접근법(마지막 철학자가 포크를 반대 순서로 집는 방식)을 사용하여 deadlock이 없는 해결책을 구현하세요.

### 연습 문제 3: Deadlock 탐지
Thread 상태를 모니터링하고 순환 대기 조건을 식별하는 deadlock 탐지기를 작성하세요.

## 요약

Deadlock은 네 가지 조건이 충족될 때 발생합니다:
1. 상호 배제(Mutual exclusion)
2. 점유 및 대기(Hold and wait)
3. 비선점(No preemption)
4. 순환 대기(Circular wait)

**예방**: 네 가지 조건 중 하나 이상을 제거
**탐지**: 자원 할당 그래프 또는 타임아웃 사용
**복구**: Thread 종료, 자원 선점, 또는 롤백

기억하세요: **가장 좋은 deadlock은 절대 발생하지 않는 deadlock입니다!**

## 추가 읽을거리

- "Operating System Concepts" - Silberschatz, Galvin, Gagne
- "The Deadlock Problem" - Coffman et al. (1971)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)

## 다음 주제

[03-livelock.md](./03-livelock.md)로 이동하여 livelock에 대해 알아보세요.
