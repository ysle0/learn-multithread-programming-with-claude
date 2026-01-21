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
