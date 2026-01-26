# 교착 상태 (Deadlock)

## 교착 상태란 무엇인가?

**교착 상태(Deadlock)**는 두 개 이상의 스레드가 서로가 보유한 자원을 기다리면서 영원히 차단되는 상황입니다. 순환 대기 조건으로 인해 어떤 스레드도 진행할 수 없어 영구적인 정체 상태가 됩니다.

### 공식적 정의

교착 상태는 다음 네 가지 조건(Coffman 조건)이 모두 동시에 성립할 때 발생합니다:

1. **상호 배제(Mutual Exclusion)**: 자원을 공유할 수 없음
2. **보유 및 대기(Hold and Wait)**: 스레드가 자원을 보유한 채 다른 자원을 기다림
3. **비선점(No Preemption)**: 자원을 강제로 빼앗을 수 없음
4. **순환 대기(Circular Wait)**: 스레드들이 자원을 기다리는 순환 고리 형성

## 시각적 표현

### 단순 2-스레드 교착 상태

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

## 네 가지 필수 조건 (Coffman 조건)

### 1. 상호 배제

자원을 공유할 수 없으며 - 한 번에 하나의 스레드만 자원을 사용할 수 있습니다.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// Only ONE thread can hold this mutex
pthread_mutex_lock(&mutex);
// Critical section - exclusive access
pthread_mutex_unlock(&mutex);
```

### 2. 보유 및 대기

최소한 하나의 자원을 보유한 스레드가 다른 스레드가 보유한 추가 자원을 획득하기 위해 대기합니다.

```c
// Thread 1 holds A and waits for B
pthread_mutex_lock(&mutex_a);  // Holding A
// ... some work ...
pthread_mutex_lock(&mutex_b);  // Waiting for B
```

### 3. 비선점

자원을 스레드로부터 강제로 제거할 수 없으며 - 자발적으로 해제되어야 합니다.

```c
// Once locked, cannot be taken away
pthread_mutex_lock(&mutex);
// ... even if higher priority thread needs it ...
pthread_mutex_unlock(&mutex);  // Must voluntarily release
```

### 4. 순환 대기

스레드의 순환 고리가 존재하며, 각 스레드는 고리의 다음 스레드가 보유한 자원을 기다립니다.

```
T1 waits for resource held by T2
T2 waits for resource held by T3
T3 waits for resource held by T1
    ↓
Circular dependency!
```

## 고전적 예제: 식사하는 철학자 문제

### 문제 설명

다섯 명의 철학자가 다섯 개의 포크가 있는 원탁에 앉아 있습니다. 각 철학자는 식사를 하려면 두 개의 포크가 필요하지만 각 철학자 사이에는 포크가 하나만 있습니다.

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

### 교착 상태 구현

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

**무슨 일이 일어나는가:**
1. 모든 철학자가 동시에 왼쪽 포크를 집습니다
2. 모든 철학자가 오른쪽 포크를 집으려고 합니다
3. 모든 오른쪽 포크는 이미 이웃의 왼쪽 포크로 보유되어 있습니다
4. **교착 상태**: 모두가 영원히 기다립니다

## 예방 전략

네 가지 Coffman 조건 중 어느 하나라도 제거하면 교착 상태를 예방할 수 있습니다.

### 전략 1: 상호 배제 제거

자원을 공유 가능하게 만듭니다 (항상 가능한 것은 아닙니다).

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

### 전략 2: 보유 및 대기 제거

모든 자원을 한 번에 획득하거나, 하나도 획득하지 않습니다.

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

**trylock을 사용한 더 나은 접근:**
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

### 전략 3: 선점 허용

타임아웃을 사용하여 대기를 포기합니다.

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

### 전략 4: 순환 대기 제거

**락 순서 지정**: 항상 일관된 전역 순서로 락을 획득합니다.

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

**락 순서를 사용한 식사하는 철학자:**
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

## 탐지 전략

### 1. 자원 할당 그래프

자원과 스레드의 그래프를 구축하여 순환을 탐지합니다.

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

### 2. 대기 그래프

자원 할당 그래프보다 간단 - 스레드 의존성만 추적합니다.

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

### 4. 타임아웃 기반 탐지

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

## 복구 전략

### 1. 스레드 종료

순환을 깨기 위해 하나 이상의 스레드를 종료합니다.

```c
// Detect deadlock then:
pthread_cancel(deadlocked_thread);
// or
pthread_kill(deadlocked_thread, SIGTERM);
```

### 2. 자원 선점

스레드가 자원을 해제하도록 강제합니다.

```c
// Difficult in practice - requires careful state management
void force_release(Thread* victim) {
    // Rollback victim's work
    // Release its resources
    // Restart victim
}
```

### 3. 롤백 및 재시작

체크포인트를 저장하고 교착 상태 탐지 시 롤백합니다.

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

## 실제 사례

### 예제 1: 데이터베이스 교착 상태

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

**해결책: 일관된 순서**
```sql
-- Always update accounts in order by ID
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Lower ID first
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Higher ID second
COMMIT;
```

### 예제 2: 파일 시스템 교착 상태

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

**해결책: 디렉토리 경로를 알파벳 순서로 잠금**
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

### 예제 3: 네트워크 프로토콜 교착 상태

```c
// Node A sends to B, waits for ACK
send_to(node_b, data);
wait_for_ack_from(node_b);

// Node B sends to A, waits for ACK
send_to(node_a, data);
wait_for_ack_from(node_a);

// Both buffers full → DEADLOCK!
```

**해결책: 타임아웃 및 재시도**
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

## 고급 패턴

### 계층적 잠금

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

### 백오프를 사용한 Try-Lock

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

### 락 프리 대안

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

## 모범 사례 요약

### 해야 할 것:
✓ 일관된 락 순서 사용
✓ 임계 영역 최소화
✓ 백오프와 함께 trylock 사용
✓ 타임아웃 구현
✓ 교착 상태 탐지 도구로 테스트
✓ 락 계층 구조 문서화
✓ 락 프리 대안 고려

### 하지 말아야 할 것:
✗ I/O를 기다리는 동안 락 보유
✗ 다른 순서로 락 획득
✗ 락을 보유한 채 알 수 없는 코드 호출
✗ 피할 수 있다면 여러 락 보유
✗ 무한정 차단

## 연습 문제

### 연습 1: 교착 상태 수정
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

### 연습 2: 안전한 식사하는 철학자 구현
비대칭 접근법을 사용하여 교착 상태 없는 솔루션을 구현하십시오 (마지막 철학자가 포크를 역순으로 집습니다).

### 연습 3: 교착 상태 탐지
스레드 상태를 모니터링하고 순환 대기 조건을 식별하는 교착 상태 탐지기를 작성하십시오.

## 요약

교착 상태는 네 가지 조건이 충족될 때 발생합니다:
1. 상호 배제
2. 보유 및 대기
3. 비선점
4. 순환 대기

**예방**: 네 가지 조건 중 최소한 하나를 제거
**탐지**: 자원 할당 그래프 또는 타임아웃 사용
**복구**: 스레드 종료, 자원 선점 또는 롤백

기억하세요: **가장 좋은 교착 상태는 결코 발생하지 않는 것입니다!**

## 추가 자료

- "Operating System Concepts" - Silberschatz, Galvin, Gagne
- "The Deadlock Problem" - Coffman et al. (1971)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)

## 다음 주제

[03-livelock.md](./03-livelock.md)로 계속하여 라이브락에 대해 배우십시오.
