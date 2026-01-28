# Deadlock

## What is Deadlock?

**Deadlock** is a situation where two or more threads are blocked forever, each waiting for resources held by the others. It's a circular waiting condition where no thread can make progress, resulting in a permanent standstill.

### Formal Definition

A deadlock occurs when ALL of the following four conditions hold simultaneously (Coffman conditions):

1. **Mutual Exclusion**: Resources cannot be shared
2. **Hold and Wait**: Threads hold resources while waiting for others
3. **No Preemption**: Resources cannot be forcibly taken away
4. **Circular Wait**: A circular chain of threads waiting for resources

## Visual Representation

### Simple Two-Thread Deadlock

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

### Resource Allocation Graph

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

## The Four Necessary Conditions (Coffman Conditions)

### 1. Mutual Exclusion

Resources cannot be shared - only one thread can use a resource at a time.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Only ONE thread can hold this mutex
pthread_mutex_lock(&mutex);
// Critical section - exclusive access
pthread_mutex_unlock(&mutex);
```

### 2. Hold and Wait

A thread holding at least one resource is waiting to acquire additional resources held by other threads.

```c
// Thread 1 holds A and waits for B
pthread_mutex_lock(&mutex_a);  // Holding A
// ... some work ...
pthread_mutex_lock(&mutex_b);  // Waiting for B
```

### 3. No Preemption

Resources cannot be forcibly removed from threads - they must be released voluntarily.

```c
// Once locked, cannot be taken away
pthread_mutex_lock(&mutex);
// ... even if higher priority thread needs it ...
pthread_mutex_unlock(&mutex);  // Must voluntarily release
```

### 4. Circular Wait

A circular chain of threads exists where each thread waits for a resource held by the next thread in the chain.

```
T1 waits for resource held by T2
T2 waits for resource held by T3
T3 waits for resource held by T1
    ↓
Circular dependency!
```

## Classic Example: Dining Philosophers Problem

### Problem Description

Five philosophers sit at a round table with five forks. Each philosopher needs TWO forks to eat but there's only one fork between each pair of philosophers.

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

### Deadlock Implementation

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
        // Think
        printf("Philosopher %d is thinking\n", id);
        sleep(1);

        // Pick up left fork
        printf("Philosopher %d picks up left fork %d\n", id, left_fork);
        pthread_mutex_lock(&forks[left_fork]);

        // Pick up right fork - DEADLOCK CAN OCCUR HERE!
        printf("Philosopher %d picks up right fork %d\n", id, right_fork);
        pthread_mutex_lock(&forks[right_fork]);

        // Eat
        printf("Philosopher %d is eating\n", id);
        sleep(2);

        // Put down forks
        pthread_mutex_unlock(&forks[right_fork]);
        pthread_mutex_unlock(&forks[left_fork]);
        printf("Philosopher %d finished eating\n", id);
    }

    return NULL;
}

int main() {
    pthread_t philosophers[NUM_PHILOSOPHERS];
    int ids[NUM_PHILOSOPHERS];

    // Initialize forks
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_mutex_init(&forks[i], NULL);
    }

    // Create philosophers
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        ids[i] = i;
        pthread_create(&philosophers[i], NULL, philosopher, &ids[i]);
    }

    // Wait forever (will deadlock)
    for (int i = 0; i < NUM_PHILOSOPHERS; i++) {
        pthread_join(philosophers[i], NULL);
    }

    return 0;
}
```

**What happens:**
1. All philosophers pick up their left fork simultaneously
2. All philosophers try to pick up their right fork
3. All right forks are already held as left forks by neighbors
4. **DEADLOCK**: Everyone waits forever

## Prevention Strategies

Breaking ANY of the four Coffman conditions prevents deadlock.

### Strategy 1: Remove Mutual Exclusion

Make resources shareable (not always possible).

```c
// Use read-write locks for read-mostly data
pthread_rwlock_t rwlock;

// Multiple readers can hold simultaneously
pthread_rwlock_rdlock(&rwlock);  // Shared access
read_data();
pthread_rwlock_unlock(&rwlock);

// Writers still need exclusive access
pthread_rwlock_wrlock(&rwlock);  // Exclusive access
write_data();
pthread_rwlock_unlock(&rwlock);
```

### Strategy 2: Remove Hold and Wait

Acquire all resources at once, or none at all.

```c
// SOLUTION: All-or-nothing resource acquisition
pthread_mutex_t global_lock = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_b = PTHREAD_MUTEX_INITIALIZER;

void critical_section() {
    // Use a global lock to acquire both mutexes atomically
    pthread_mutex_lock(&global_lock);

    pthread_mutex_lock(&mutex_a);
    pthread_mutex_lock(&mutex_b);

    pthread_mutex_unlock(&global_lock);

    // Work with both resources
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

**Better approach with trylock:**
```c
#include <pthread.h>
#include <stdbool.h>
#include <time.h>

bool acquire_both(pthread_mutex_t* m1, pthread_mutex_t* m2) {
    pthread_mutex_lock(m1);

    if (pthread_mutex_trylock(m2) == 0) {
        return true;  // Got both locks
    }

    // Couldn't get second lock, release first
    pthread_mutex_unlock(m1);
    return false;  // Failed to acquire both
}

void critical_section() {
    while (!acquire_both(&mutex_a, &mutex_b)) {
        // Back off and retry
        struct timespec ts = {0, 100000};  // 100 microseconds
        nanosleep(&ts, NULL);
    }

    // Work with both resources
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

### Strategy 3: Allow Preemption

Use timeouts to abandon waiting.

```c
#include <pthread.h>
#include <time.h>
#include <errno.h>

void critical_section_with_timeout() {
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 1;  // 1 second timeout

    pthread_mutex_lock(&mutex_a);

    int result = pthread_mutex_timedlock(&mutex_b, &timeout);

    if (result == ETIMEDOUT) {
        // Couldn't acquire, release and retry
        pthread_mutex_unlock(&mutex_a);
        printf("Timeout! Backing off...\n");
        return;
    }

    // Work with both resources
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

### Strategy 4: Remove Circular Wait

**Lock Ordering**: Always acquire locks in a consistent global order.

```c
// SOLUTION: Ordered lock acquisition
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t mutex_a = PTHREAD_MUTEX_INITIALIZER;
pthread_mutex_t mutex_b = PTHREAD_MUTEX_INITIALIZER;

void thread1_work() {
    // Always lock A before B
    pthread_mutex_lock(&mutex_a);
    printf("Thread 1: Locked A\n");

    pthread_mutex_lock(&mutex_b);
    printf("Thread 1: Locked B\n");

    // Critical section
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}

void thread2_work() {
    // Always lock A before B (same order!)
    pthread_mutex_lock(&mutex_a);
    printf("Thread 2: Locked A\n");

    pthread_mutex_lock(&mutex_b);
    printf("Thread 2: Locked B\n");

    // Critical section
    // ...

    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
}
```

**Dining Philosophers with Lock Ordering:**
```c
void* philosopher_ordered(void* arg) {
    int id = *(int*)arg;
    int left_fork = id;
    int right_fork = (id + 1) % NUM_PHILOSOPHERS;

    // SOLUTION: Always pick up lower-numbered fork first
    int first_fork = (left_fork < right_fork) ? left_fork : right_fork;
    int second_fork = (left_fork < right_fork) ? right_fork : left_fork;

    while (1) {
        printf("Philosopher %d is thinking\n", id);
        sleep(1);

        // Pick up forks in order
        pthread_mutex_lock(&forks[first_fork]);
        pthread_mutex_lock(&forks[second_fork]);

        // Eat
        printf("Philosopher %d is eating\n", id);
        sleep(2);

        // Put down forks
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

## Detection Strategies

### 1. Resource Allocation Graph

Build a graph of resources and threads to detect cycles.

```c
typedef struct {
    int thread_id;
    int* held_resources;
    int* waiting_for;
} ThreadInfo;

bool detect_cycle(ThreadInfo* threads, int num_threads) {
    // Use DFS to detect cycle in wait-for graph
    // If cycle found, deadlock exists
    // Implementation: graph traversal algorithm
}
```

### 2. Wait-For Graph

Simpler than resource allocation graph - only tracks thread dependencies.

```
Thread Dependencies:
T1 → T2  (T1 waits for T2)
T2 → T3  (T2 waits for T3)
T3 → T1  (T3 waits for T1)

Cycle: T1 → T2 → T3 → T1
Result: DEADLOCK DETECTED
```

### 3. Runtime Detection Tools

```bash
# Using Helgrind (Valgrind)
valgrind --tool=helgrind ./program

# Sample output:
# Thread #1: lock order "0x4C0D040 before 0x4C0D080" violated
# Thread #2: lock order "0x4C0D080 before 0x4C0D040" violated
# => Possible deadlock detected

# Using GDB to inspect deadlocked program
gdb -p <pid>
(gdb) info threads
(gdb) thread apply all bt  # Backtrace all threads
```

### 4. Timeout-Based Detection

```c
#include <pthread.h>
#include <time.h>
#include <stdio.h>

void detect_with_timeout() {
    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += 5;  // 5 second timeout

    int result = pthread_mutex_timedlock(&mutex, &timeout);

    if (result == ETIMEDOUT) {
        printf("DEADLOCK suspected: timeout after 5 seconds\n");
        // Take corrective action
    }
}
```

## Recovery Strategies

### 1. Thread Termination

Kill one or more threads to break the cycle.

```c
// Detect deadlock then:
pthread_cancel(deadlocked_thread);
// or
pthread_kill(deadlocked_thread, SIGTERM);
```

### 2. Resource Preemption

Force a thread to release resources.

```c
// Difficult in practice - requires careful state management
void force_release(Thread* victim) {
    // Rollback victim's work
    // Release its resources
    // Restart victim
}
```

### 3. Rollback and Restart

Save checkpoints and rollback on deadlock detection.

```c
typedef struct {
    void* saved_state;
    pthread_mutex_t* held_locks;
} Checkpoint;

void rollback_on_deadlock(Checkpoint* cp) {
    // Release all locks
    for (int i = 0; i < cp->num_locks; i++) {
        pthread_mutex_unlock(&cp->held_locks[i]);
    }

    // Restore state
    restore_state(cp->saved_state);

    // Retry operation
}
```

## Real-World Examples

### Example 1: Database Deadlock

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

// DEADLOCK! T1 waits for T2, T2 waits for T1
```

**Solution: Consistent ordering**
```sql
-- Always update accounts in order by ID
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lower ID first
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Higher ID second
COMMIT;
```

### Example 2: File System Deadlock

```c
// Thread 1: Move file from /a to /b
lock_directory("/a");
lock_directory("/b");
move_file("/a/file.txt", "/b/file.txt");
unlock_directory("/b");
unlock_directory("/a");

// Thread 2: Move file from /b to /a
lock_directory("/b");  // Locks in opposite order!
lock_directory("/a");
move_file("/b/other.txt", "/a/other.txt");
unlock_directory("/a");
unlock_directory("/b");

// DEADLOCK possible!
```

**Solution: Lock directory paths in alphabetical order**
```c
void move_file_safe(const char* from_dir, const char* to_dir,
                   const char* filename) {
    // Determine lock order
    const char* first = (strcmp(from_dir, to_dir) < 0) ? from_dir : to_dir;
    const char* second = (strcmp(from_dir, to_dir) < 0) ? to_dir : from_dir;

    lock_directory(first);
    lock_directory(second);

    // Perform move
    char from_path[256], to_path[256];
    snprintf(from_path, sizeof(from_path), "%s/%s", from_dir, filename);
    snprintf(to_path, sizeof(to_path), "%s/%s", to_dir, filename);
    rename(from_path, to_path);

    unlock_directory(second);
    unlock_directory(first);
}
```

### Example 3: Network Protocol Deadlock

```c
// Node A sends to B, waits for ACK
send_to(node_b, data);
wait_for_ack_from(node_b);

// Node B sends to A, waits for ACK
send_to(node_a, data);
wait_for_ack_from(node_a);

// Both buffers full → DEADLOCK!
```

**Solution: Timeout and retry**
```c
bool send_with_timeout(Node* target, Data* data, int timeout_ms) {
    send_to(target, data);

    struct timespec timeout;
    clock_gettime(CLOCK_REALTIME, &timeout);
    timeout.tv_sec += timeout_ms / 1000;

    if (wait_for_ack_timeout(target, &timeout) == ETIMEDOUT) {
        return false;  // Timeout, no deadlock
    }

    return true;
}
```

## Advanced Patterns

### Hierarchical Locking

```c
// Define lock hierarchy levels
#define LEVEL_DATABASE  100
#define LEVEL_TABLE     200
#define LEVEL_ROW       300

typedef struct {
    pthread_mutex_t mutex;
    int level;
} HierarchicalMutex;

// Can only acquire locks in increasing level order
void hierarchical_lock(HierarchicalMutex* m, int current_level) {
    if (m->level <= current_level) {
        fprintf(stderr, "Lock ordering violation!\n");
        abort();
    }
    pthread_mutex_lock(&m->mutex);
}
```

### Try-Lock with Backoff

```c
#include <pthread.h>
#include <unistd.h>
#include <stdbool.h>

bool try_acquire_with_backoff(pthread_mutex_t* m1, pthread_mutex_t* m2) {
    int backoff = 1000;  // Start with 1ms

    for (int attempts = 0; attempts < 10; attempts++) {
        pthread_mutex_lock(m1);

        if (pthread_mutex_trylock(m2) == 0) {
            return true;  // Success!
        }

        // Failed, release and backoff
        pthread_mutex_unlock(m1);
        usleep(backoff);
        backoff *= 2;  // Exponential backoff
    }

    return false;  // Give up after 10 attempts
}
```

### Lock-Free Alternative

```c
// Avoid deadlock entirely with lock-free structures
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

// No locks → No deadlock possible!
```

## Best Practices Summary

### DO:
✓ Use consistent lock ordering
✓ Minimize critical sections
✓ Use trylock with backoff
✓ Implement timeouts
✓ Test with deadlock detection tools
✓ Document lock hierarchies
✓ Consider lock-free alternatives

### DON'T:
✗ Hold locks while waiting for I/O
✗ Acquire locks in different orders
✗ Call unknown code while holding locks
✗ Hold multiple locks if avoidable
✗ Block indefinitely

## Exercises

### Exercise 1: Fix the Deadlock
```c
void transfer(Account* from, Account* to, int amount) {
    pthread_mutex_lock(&from->mutex);
    pthread_mutex_lock(&to->mutex);

    from->balance -= amount;
    to->balance += amount;

    pthread_mutex_unlock(&to->mutex);
    pthread_mutex_unlock(&from->mutex);
}

// This can deadlock! Fix it.
```

### Exercise 2: Implement Safe Dining Philosophers
Implement a deadlock-free solution using an asymmetric approach (last philosopher picks up forks in reverse order).

### Exercise 3: Detect Deadlock
Write a deadlock detector that monitors thread states and identifies circular wait conditions.

## Summary

Deadlock occurs when four conditions are met:
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

**Prevention**: Break at least one of the four conditions
**Detection**: Use resource allocation graphs or timeouts
**Recovery**: Terminate threads, preempt resources, or rollback

Remember: **The best deadlock is the one that never happens!**

## Further Reading

- "Operating System Concepts" - Silberschatz, Galvin, Gagne
- "The Deadlock Problem" - Coffman et al. (1971)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)

## Next Topic

Continue to [03-livelock.md](./03-livelock.md) to learn about livelock.
