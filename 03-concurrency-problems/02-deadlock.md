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
