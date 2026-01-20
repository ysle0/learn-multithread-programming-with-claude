# Starvation

## What is Starvation?

**Starvation** occurs when a thread is perpetually denied access to resources it needs to make progress. Unlike deadlock where all threads are stuck, in starvation some threads make progress while others are indefinitely delayed. The starved thread may eventually get the resource, but the wait time is unbounded and unpredictable.

### Formal Definition

A thread suffers from starvation when:
1. It is ready to execute and needs resources
2. Other threads continuously acquire those resources
3. The thread waits indefinitely without making progress
4. The system as a whole makes progress (unlike deadlock)

## Visual Representation

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

### Starvation vs Other Problems

```
┌───────────────┬──────────┬──────────┬────────────┬──────────┐
│   Problem     │  Blocked │ Progress │   Cause    │ Severity │
├───────────────┼──────────┼──────────┼────────────┼──────────┤
│ Deadlock      │   All    │   None   │  Circular  │ Critical │
│ Livelock      │   None   │   None   │  Collision │   High   │
│ Starvation    │   Some   │  Partial │  Unfair    │  Medium  │
└───────────────┴──────────┴──────────┴────────────┴──────────┘
```

## Common Causes of Starvation

### 1. Priority-Based Scheduling

High-priority threads always preempt low-priority threads.

```c
#include <pthread.h>
#include <stdio.h>
#include <sched.h>

pthread_mutex_t resource = PTHREAD_MUTEX_INITIALIZER;

void* high_priority_thread(void* arg) {
    // Set high priority
    struct sched_param param;
    param.sched_priority = 99;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("High priority: Working\n");
        // Do work...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

void* low_priority_thread(void* arg) {
    // Set low priority
    struct sched_param param;
    param.sched_priority = 1;
    pthread_setschedparam(pthread_self(), SCHED_FIFO, &param);

    while (1) {
        pthread_mutex_lock(&resource);
        printf("Low priority: Working\n");  // MAY NEVER PRINT
        // Do work...
        pthread_mutex_unlock(&resource);
    }
    return NULL;
}

// Low priority thread can STARVE if high priority runs continuously
```

### 2. Unfair Lock Implementation

Some lock implementations don't guarantee fairness.

```c
// Unfair mutex implementation (simplified)
typedef struct {
    atomic_int locked;
    // No queue - threads race to acquire
} UnfairMutex;

void unfair_lock(UnfairMutex* m) {
    // Spin until successful
    while (1) {
        int expected = 0;
        if (atomic_compare_exchange_weak(&m->locked, &expected, 1)) {
            return;  // Acquired
        }
        // Some threads might retry faster than others!
        // Fast threads can starve slow ones
    }
}

// Thread with faster CPU core might always win
// Thread with slower core might STARVE
```

### 3. Reader-Writer Problem

Writers can starve if readers keep arriving.

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

    // Wait for all readers to finish
    while (lock->readers > 0) {
        pthread_mutex_unlock(&lock->mutex);
        sched_yield();
        pthread_mutex_lock(&lock->mutex);
    }

    // Now have write access
}

// PROBLEM: If readers keep arriving, writer STARVES
```

**Timeline:**
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

### 4. Producer-Consumer with Unfair Semaphore

```c
#include <semaphore.h>
#include <pthread.h>

#define BUFFER_SIZE 10

sem_t empty;  // Count of empty slots
sem_t full;   // Count of full slots

void* producer(void* arg) {
    while (1) {
        sem_wait(&empty);  // Wait for empty slot

        // Produce item
        produce_item();

        sem_post(&full);   // Signal item available
    }
    return NULL;
}

void* consumer(void* arg) {
    while (1) {
        sem_wait(&full);   // Wait for item

        // Consume item
        consume_item();

        sem_post(&empty);  // Signal slot empty
    }
    return NULL;
}

// If many fast producers and one slow consumer,
// slow consumer might STARVE
```

## Examples by Category

### Example 1: Thread Pool Starvation

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

#define NUM_WORKERS 4
#define QUEUE_SIZE 100

typedef struct {
    void (*function)(void*);
    void* arg;
    int priority;  // Higher = more important
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

    // Insert by priority (higher priority first)
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

        // Take highest priority task
        Task task = pool.queue[0];
        pool.size--;

        // Shift queue
        for (int i = 0; i < pool.size; i++) {
            pool.queue[i] = pool.queue[i+1];
        }

        pthread_mutex_unlock(&pool.mutex);

        // Execute task
        task.function(task.arg);
    }
    return NULL;
}

// PROBLEM: Low priority tasks can STARVE if high priority
// tasks keep arriving
```

**Visualization:**
```
Queue State (priority-ordered):
[9][9][9][8][8][7][7][7][6][5] ← High priority kept arriving
                               [2] ← Low priority task STARVING
```

### Example 2: Disk I/O Scheduler Starvation

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int track;      // Disk track number
    int timestamp;  // When request arrived
} IORequest;

// SCAN (Elevator) algorithm - can cause starvation
void scan_schedule(IORequest* requests, int count, int current_track) {
    int direction = 1;  // 1 = up, -1 = down

    while (1) {
        int served = 0;

        // Serve requests in current direction
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
            direction = -direction;  // Reverse direction
        }

        // PROBLEM: Requests at far end can STARVE
        // if new requests keep arriving on current side
    }
}
```

### Example 3: Network Packet Processing

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

        // Always process highest priority first
        int highest_idx = 0;
        for (int i = 1; i < queue_size; i++) {
            if (queue[i].priority > queue[highest_idx].priority) {
                highest_idx = i;
            }
        }

        Packet p = queue[highest_idx];

        // Check for starvation
        uint64_t wait_time = current_time() - p.arrival_time;
        if (wait_time > 10000) {  // 10 seconds
            printf("WARNING: Packet starved for %llu ms\n", wait_time);
        }

        process_packet(&p);

        // Remove from queue
        queue[highest_idx] = queue[--queue_size];
    }
}

// Low priority packets STARVE if high priority keep arriving
```

## Solutions and Prevention

### Solution 1: Fair Mutex (FIFO Order)

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
        return;  // Got lock immediately
    }

    // Add to wait queue
    if (fm->tail) {
        fm->tail->next = &node;
    } else {
        fm->head = &node;
    }
    fm->tail = &node;

    // Wait for our turn
    while (!node.ready) {
        pthread_cond_wait(&node.cond, &fm->mutex);
    }

    pthread_mutex_unlock(&fm->mutex);
    pthread_cond_destroy(&node.cond);
}

void fair_mutex_unlock(FairMutex* fm) {
    pthread_mutex_lock(&fm->mutex);

    if (fm->head) {
        // Wake next waiter
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

// FIFO ordering prevents starvation
```

### Solution 2: Fair Reader-Writer Lock

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

    // Wait if there's a writer or waiting writers
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
        // Wake a waiting writer
        pthread_cond_signal(&lock->writers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

void fair_write_lock(FairRWLock* lock) {
    pthread_mutex_lock(&lock->mutex);
    lock->waiting_writers++;

    // Wait for readers and writers to finish
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
        // Prefer waiting writers
        pthread_cond_signal(&lock->writers_cond);
    } else {
        // Wake all waiting readers
        pthread_cond_broadcast(&lock->readers_cond);
    }

    pthread_mutex_unlock(&lock->mutex);
}

// Writers won't starve - they're preferred after current readers
```

### Solution 3: Aging Priority

Increase priority of waiting threads over time.

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
    // Increase priority by 1 every 10 seconds
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

// Old low-priority tasks eventually become high priority
// Prevents indefinite starvation
```

### Solution 4: Round-Robin Scheduling

Give each thread a time slice.

```c
#include <pthread.h>
#include <signal.h>
#include <time.h>

#define NUM_THREADS 5
#define TIME_SLICE_MS 100

pthread_t threads[NUM_THREADS];
int current_thread = 0;

void switch_thread(int sig) {
    // Pause current thread
    pthread_kill(threads[current_thread], SIGSTOP);

    // Switch to next thread
    current_thread = (current_thread + 1) % NUM_THREADS;

    // Resume next thread
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

// All threads get equal CPU time - no starvation
```

### Solution 5: Two-Level Feedback Queue

```c
#define NUM_QUEUES 3

typedef struct {
    Task queues[NUM_QUEUES][100];
    int sizes[NUM_QUEUES];
    int execution_counts[1000];  // Track per-thread
} FeedbackQueue;

FeedbackQueue fbq = {{{0}}, {0}, {0}};

void enqueue_with_feedback(int thread_id, Task task) {
    // New tasks start at highest priority queue
    int queue_level = 0;

    // Demote if executed too many times
    int exec_count = fbq.execution_counts[thread_id];
    if (exec_count > 10) queue_level = 2;      // Low priority
    else if (exec_count > 3) queue_level = 1;  // Medium priority

    fbq.queues[queue_level][fbq.sizes[queue_level]++] = task;
}

Task* get_next_with_feedback() {
    // Service higher priority queues first
    for (int level = 0; level < NUM_QUEUES; level++) {
        if (fbq.sizes[level] > 0) {
            Task* task = &fbq.queues[level][0];

            // Remove from queue
            for (int i = 0; i < fbq.sizes[level] - 1; i++) {
                fbq.queues[level][i] = fbq.queues[level][i + 1];
            }
            fbq.sizes[level]--;

            return task;
        }
    }
    return NULL;
}

// Even low-priority tasks get served when high queue is empty
```

## Detection Strategies

### 1. Wait Time Monitoring

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

### 2. Fairness Metrics

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

### 3. Queue Length Tracking

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

## Fairness Concepts

### Strong Fairness

Every thread that wants access will eventually get it.

```c
// Example: FIFO mutex (shown earlier)
// Guarantees: If thread requests lock, it WILL get it
```

### Weak Fairness

If a thread keeps wanting access, it will eventually get it.

```c
// Example: Simple mutex with no guarantees
// Only ensures: continuous requests eventually succeed
```

### No Fairness

No guarantees about who gets access when.

```c
// Example: Spinlock without queue
while (!atomic_compare_exchange(&lock, &expected, 1)) {
    // Any thread might win - no fairness
}
```

### Fairness Comparison

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

## Best Practices

### DO:
- ✓ Use fair synchronization primitives
- ✓ Monitor wait times and detect starvation
- ✓ Implement aging for priority systems
- ✓ Bound priority ranges
- ✓ Use FIFO queues where possible
- ✓ Set timeout limits
- ✓ Test under high load conditions

### DON'T:
- ✗ Use unbounded priorities
- ✗ Always prefer one class of threads
- ✗ Ignore wait time metrics
- ✗ Assume fairness without verification
- ✗ Use pure priority scheduling for long-running tasks

## Summary

**Starvation** occurs when threads are perpetually denied resources due to:
- Unfair scheduling
- Priority schemes
- Reader-writer imbalance
- Lack of fairness guarantees

**Key Differences:**
```
Deadlock:    No progress by anyone
Livelock:    Activity but no progress
Starvation:  Some progress, but not by everyone
```

**Prevention Strategies:**
1. Fair locks (FIFO ordering)
2. Aging algorithms
3. Bounded waiting
4. Round-robin scheduling
5. Fairness monitoring

## Exercises

### Exercise 1: Detect Starvation
Add monitoring to detect when a thread has waited more than 5 seconds.

### Exercise 2: Implement Fair Queue
Create a fair priority queue where old low-priority items eventually get served.

### Exercise 3: Fix Reader Starvation
Modify the reader-writer lock to prevent writer starvation.

## Further Reading

- "Operating Systems: Three Easy Pieces" - Remzi Arpaci-Dusseau
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Modern Operating Systems" - Andrew Tanenbaum

## Next Topic

Continue to [05-priority-inversion.md](./05-priority-inversion.md) to learn about priority inversion.
