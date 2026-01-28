# Race Conditions

## What is a Race Condition?

A **race condition** occurs when the behavior of a program depends on the relative timing or interleaving of multiple threads. When two or more threads access shared data concurrently, and at least one thread modifies the data, the final result becomes unpredictable and depends on which thread "wins the race."

### Formal Definition

A race condition exists when:
1. Two or more threads access the same memory location
2. At least one access is a write operation
3. The accesses are not synchronized
4. The result depends on the timing of execution

## Visual Representation

### Non-Deterministic Execution

```
Thread 1                Thread 2                Shared Memory
--------                --------                -------------
                                                counter = 0

Read counter (0)
                        Read counter (0)
Increment (0 + 1)
                        Increment (0 + 1)
Write counter = 1
                        Write counter = 1
                                                counter = 1 (Wrong!)

Expected: counter = 2
Actual: counter = 1
```

### Timeline Diagram

```
Time ─────────────────────────────────────────────────────▶

Thread 1: [─R─][─+─][─W─]      [─R─][─+─][─W─]
Thread 2:      [─R─][─+─][─W─]      [─R─][─+─][─W─]

Legend: R=Read, +=Compute, W=Write

Overlapping operations cause race condition!
```

## Types of Race Conditions

### 1. Read-Modify-Write Race

The most common type, where multiple threads read, modify, and write back a value.

```c
// PROBLEM: Race condition on counter
#include <pthread.h>
#include <stdio.h>

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // NOT ATOMIC: read, increment, write
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Counter: %d (expected: 2000000)\n", counter);
    // Output varies: 1000000, 1500000, 1850000, etc.
    return 0;
}
```

**Why it happens:**
```assembly
; counter++ compiles to multiple instructions:
MOV  eax, [counter]   ; Read current value
INC  eax              ; Increment in register
MOV  [counter], eax   ; Write back to memory

; Thread interleaving can happen between ANY of these!
```

**SOLUTION 1: Using Mutex**
```c
#include <pthread.h>
#include <stdio.h>

int counter = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        pthread_mutex_lock(&mutex);
        counter++;
        pthread_mutex_unlock(&mutex);
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Counter: %d (expected: 2000000)\n", counter);
    // Output: Always 2000000

    pthread_mutex_destroy(&mutex);
    return 0;
}
```

**SOLUTION 2: Using Atomic Operations**
```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

atomic_int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1);  // Atomic operation
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Counter: %d (expected: 2000000)\n", counter);
    // Output: Always 2000000
    return 0;
}
```

### 2. Check-Then-Act Race

A thread checks a condition and then acts based on that check, but the condition can change between the check and the act.

```c
// PROBLEM: Check-then-act race condition
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int balance;
} BankAccount;

BankAccount account = {1000};

void* withdraw(void* arg) {
    int amount = *(int*)arg;

    // CHECK
    if (account.balance >= amount) {
        // Context switch can happen here!
        // ACT
        account.balance -= amount;
        printf("Withdrew %d, balance: %d\n", amount, account.balance);
    } else {
        printf("Insufficient funds\n");
    }

    return NULL;
}

int main() {
    pthread_t t1, t2;
    int amount1 = 600, amount2 = 600;

    pthread_create(&t1, NULL, withdraw, &amount1);
    pthread_create(&t2, NULL, withdraw, &amount2);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final balance: %d (expected: >= 0)\n", account.balance);
    // Can output: Final balance: -200 (OVERDRAFT!)
    return 0;
}
```

**Timeline of the Bug:**
```
Initial Balance: $1000

Thread 1                    Thread 2                    Balance
--------                    --------                    -------
Check: balance >= 600 ✓
                            Check: balance >= 600 ✓
Withdraw 600
                            Withdraw 600
                                                        -200 (BUG!)
```

**SOLUTION: Atomic Check-and-Act**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>

typedef struct {
    int balance;
    pthread_mutex_t mutex;
} BankAccount;

BankAccount account = {1000, PTHREAD_MUTEX_INITIALIZER};

bool withdraw(int amount) {
    pthread_mutex_lock(&account.mutex);

    bool success = false;
    if (account.balance >= amount) {
        account.balance -= amount;
        success = true;
        printf("Withdrew %d, balance: %d\n", amount, account.balance);
    } else {
        printf("Insufficient funds\n");
    }

    pthread_mutex_unlock(&account.mutex);
    return success;
}

void* withdraw_thread(void* arg) {
    withdraw(*(int*)arg);
    return NULL;
}

int main() {
    pthread_t t1, t2;
    int amount1 = 600, amount2 = 600;

    pthread_create(&t1, NULL, withdraw_thread, &amount1);
    pthread_create(&t2, NULL, withdraw_thread, &amount2);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Final balance: %d\n", account.balance);
    // Output: Always >= 0

    pthread_mutex_destroy(&account.mutex);
    return 0;
}
```

### 3. Lazy Initialization Race (Double-Checked Locking)

A subtle race condition that occurs in singleton patterns.

```c
// PROBLEM: Broken double-checked locking
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    int data;
} Singleton;

Singleton* instance = NULL;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

Singleton* get_instance() {
    if (instance == NULL) {  // First check (UNSYNCHRONIZED)
        pthread_mutex_lock(&mutex);
        if (instance == NULL) {  // Second check
            instance = malloc(sizeof(Singleton));
            instance->data = 42;  // Initialization
        }
        pthread_mutex_unlock(&mutex);
    }
    return instance;
}

// BUG: Compiler/CPU can reorder:
// 1. Allocate memory
// 2. Assign to instance
// 3. Initialize data
// Thread 2 might see non-NULL but uninitialized instance!
```

**SOLUTION: Using Atomic Operations**
```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdlib.h>

typedef struct {
    int data;
} Singleton;

atomic_uintptr_t instance = 0;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

Singleton* get_instance() {
    Singleton* tmp = (Singleton*)atomic_load(&instance);

    if (tmp == NULL) {
        pthread_mutex_lock(&mutex);
        tmp = (Singleton*)atomic_load(&instance);
        if (tmp == NULL) {
            tmp = malloc(sizeof(Singleton));
            tmp->data = 42;
            atomic_store(&instance, (uintptr_t)tmp);
        }
        pthread_mutex_unlock(&mutex);
    }

    return tmp;
}
```

## Real-World Examples

### Example 1: Thread-Safe Stack

```c
// PROBLEM: Race condition in stack operations
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>

#define MAX_SIZE 100

typedef struct {
    int items[MAX_SIZE];
    int top;
} Stack;

Stack stack = {{0}, -1};

bool push(int value) {
    if (stack.top >= MAX_SIZE - 1) return false;

    stack.top++;              // RACE 1
    stack.items[stack.top] = value;  // RACE 2
    return true;
}

bool pop(int* value) {
    if (stack.top < 0) return false;

    *value = stack.items[stack.top];  // RACE 1
    stack.top--;              // RACE 2
    return true;
}

// Multiple threads calling push/pop creates chaos!
```

**SOLUTION: Lock-Based Thread-Safe Stack**
```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>

#define MAX_SIZE 100

typedef struct {
    int items[MAX_SIZE];
    int top;
    pthread_mutex_t mutex;
} ThreadSafeStack;

void stack_init(ThreadSafeStack* s) {
    s->top = -1;
    pthread_mutex_init(&s->mutex, NULL);
}

bool stack_push(ThreadSafeStack* s, int value) {
    pthread_mutex_lock(&s->mutex);

    bool success = false;
    if (s->top < MAX_SIZE - 1) {
        s->top++;
        s->items[s->top] = value;
        success = true;
    }

    pthread_mutex_unlock(&s->mutex);
    return success;
}

bool stack_pop(ThreadSafeStack* s, int* value) {
    pthread_mutex_lock(&s->mutex);

    bool success = false;
    if (s->top >= 0) {
        *value = s->items[s->top];
        s->top--;
        success = true;
    }

    pthread_mutex_unlock(&s->mutex);
    return success;
}

void stack_destroy(ThreadSafeStack* s) {
    pthread_mutex_destroy(&s->mutex);
}
```

### Example 2: Reference Counting

```c
// PROBLEM: Race condition in reference counting
typedef struct {
    int* data;
    int ref_count;
} SharedObject;

void acquire(SharedObject* obj) {
    obj->ref_count++;  // RACE CONDITION!
}

void release(SharedObject* obj) {
    obj->ref_count--;  // RACE CONDITION!
    if (obj->ref_count == 0) {
        free(obj->data);
        free(obj);
    }
}
```

**SOLUTION: Atomic Reference Counting**
```c
#include <stdatomic.h>
#include <stdlib.h>

typedef struct {
    int* data;
    atomic_int ref_count;
} SharedObject;

void acquire(SharedObject* obj) {
    atomic_fetch_add(&obj->ref_count, 1);
}

void release(SharedObject* obj) {
    if (atomic_fetch_sub(&obj->ref_count, 1) == 1) {
        // We were the last reference
        free(obj->data);
        free(obj);
    }
}
```

## Internal Mechanisms

### ThreadSanitizer (TSan) 내부 동작

ThreadSanitizer는 컴파일러 계측(instrumentation)을 통해 데이터 레이스를 탐지합니다.

#### Shadow Memory 구조

```
┌─────────────────────────────────────────────────────────────┐
│                     Application Memory                       │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐        │
│  │ 8B  │ 8B  │ 8B  │ 8B  │ 8B  │ 8B  │ 8B  │ 8B  │        │
│  └──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘        │
│     │     │     │     │     │     │     │     │            │
│     ▼     ▼     ▼     ▼     ▼     ▼     ▼     ▼            │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐        │
│  │Shadow│Shadow│Shadow│Shadow│Shadow│Shadow│Shadow│Shadow│ │
│  │ Cell │ Cell │ Cell │ Cell │ Cell │ Cell │ Cell │ Cell │ │
│  │ 32B  │ 32B  │ 32B  │ 32B  │ 32B  │ 32B  │ 32B  │ 32B  │ │
│  └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘        │
│                     Shadow Memory (8x larger)               │
└─────────────────────────────────────────────────────────────┘
```

#### Shadow Cell 구조 (각 8바이트 앱 메모리당)

```c
// TSan shadow cell: 4개의 shadow word (각 8바이트)
struct ShadowCell {
    ShadowWord words[4];  // 최근 4개 접근 기록
};

struct ShadowWord {
    // 64-bit packed format:
    // [TID:16][Epoch:42][IsWrite:1][AccessSize:2][Offset:3]
    uint16_t tid;         // Thread ID (최대 65535개 스레드)
    uint64_t epoch : 42;  // Vector clock epoch
    uint8_t  is_write : 1;
    uint8_t  size : 2;    // 1, 2, 4, 8 bytes
    uint8_t  offset : 3;  // 8바이트 내 오프셋 (0-7)
};
```

#### Happens-Before 관계 추적

```c
// TSan이 추적하는 동기화 이벤트들
void tsan_mutex_lock(void* mutex) {
    // mutex의 release epoch와 현재 스레드의 clock 동기화
    ThreadState* thr = get_current_thread();
    MutexInfo* m = get_mutex_info(mutex);

    // Acquire semantics: 이전 holder의 clock을 가져옴
    thr->clock.acquire(m->release_clock);
}

void tsan_mutex_unlock(void* mutex) {
    ThreadState* thr = get_current_thread();
    MutexInfo* m = get_mutex_info(mutex);

    // Release semantics: 현재 clock을 mutex에 저장
    m->release_clock.release(thr->clock);
    thr->clock.tick();  // epoch 증가
}

// 메모리 접근 시 레이스 체크
void tsan_memory_access(void* addr, int size, bool is_write) {
    ShadowCell* shadow = addr_to_shadow(addr);
    ThreadState* thr = get_current_thread();

    for (int i = 0; i < 4; i++) {
        ShadowWord prev = shadow->words[i];

        // 다른 스레드의 접근이고, 둘 중 하나가 write이고,
        // happens-before 관계가 없으면 = RACE!
        if (prev.tid != thr->tid &&
            (prev.is_write || is_write) &&
            !thr->clock.happens_after(prev.tid, prev.epoch)) {

            report_race(addr, prev, thr);
        }
    }

    // 현재 접근 기록 (가장 오래된 것 교체)
    shadow->words[oldest_idx] = make_shadow_word(thr, is_write, size);
}
```

### 하드웨어 수준 가시성 문제

데이터 레이스는 CPU 캐시 일관성 문제와 밀접하게 관련됩니다.

#### Store Buffer와 가시성

```
┌─────────────────────────────────────────────────────────────┐
│                    CPU 0                    CPU 1           │
│  ┌─────────────┐                      ┌─────────────┐      │
│  │   Core 0    │                      │   Core 1    │      │
│  │  ┌───────┐  │                      │  ┌───────┐  │      │
│  │  │ Load  │  │                      │  │ Load  │  │      │
│  │  │ Queue │  │                      │  │ Queue │  │      │
│  │  └───────┘  │                      │  └───────┘  │      │
│  │      │      │                      │      │      │      │
│  │  ┌───────┐  │                      │  ┌───────┐  │      │
│  │  │ Store │  │  ← 다른 CPU에서      │  │ Store │  │      │
│  │  │Buffer │  │    안 보임!          │  │Buffer │  │      │
│  │  └───┬───┘  │                      │  └───┬───┘  │      │
│  └──────┼──────┘                      └──────┼──────┘      │
│         │                                    │              │
│         ▼                                    ▼              │
│  ┌──────────────────────────────────────────────────┐      │
│  │              L3 Cache (Shared)                   │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘

문제 시나리오:
  CPU 0: x = 1    (Store Buffer에 저장, 아직 캐시에 반영 안됨)
  CPU 1: r = x    (캐시에서 읽음 = 0, CPU 0의 store를 못 봄)
```

#### MESI 프로토콜과 레이스

```c
// 데이터 레이스가 발생하는 MESI 상태 전이 예시
/*
시간    CPU 0 Action       CPU 0 State    CPU 1 Action       CPU 1 State
────    ──────────────    ───────────    ──────────────    ───────────
 1      Read X            Shared (S)     -                  Invalid (I)
 2      -                 Shared (S)     Read X             Shared (S)
 3      Write X=1         Modified (M)   -                  Invalid (I)
 4      -                 Modified (M)   Read X (stale!)    ← RACE!

동기화 없이는 CPU 1이 stale 값을 읽을 수 있음
*/
```

### 컴파일러 최적화와 레이스

```c
// 원본 코드
int ready = 0;
int data = 0;

// Thread 1
void producer() {
    data = 42;
    ready = 1;
}

// 컴파일러가 재배치 가능:
void producer_reordered() {
    ready = 1;     // 먼저 실행될 수 있음!
    data = 42;
}

// Thread 2
void consumer() {
    while (ready == 0);
    use(data);     // data가 42가 아닐 수 있음!
}

// 해결: volatile이 아닌 atomic 사용
#include <stdatomic.h>
atomic_int ready = 0;
int data = 0;

void producer_safe() {
    data = 42;
    atomic_store_explicit(&ready, 1, memory_order_release);
}

void consumer_safe() {
    while (atomic_load_explicit(&ready, memory_order_acquire) == 0);
    use(data);     // data = 42 보장
}
```

### Helgrind vs TSan 비교

```
┌─────────────────┬──────────────────────┬──────────────────────┐
│     Feature     │      Helgrind        │    ThreadSanitizer   │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 구현 방식       │ 바이너리 계측        │ 컴파일러 계측        │
│                 │ (Valgrind)           │ (-fsanitize=thread)  │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 오버헤드        │ 20-100x 느림         │ 2-20x 느림           │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 메모리 사용     │ ~2x                  │ ~5-10x (shadow mem)  │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 정확도          │ Lockset algorithm    │ Happens-before       │
│                 │ (false positives     │ (더 정확)            │
│                 │  가능)               │                      │
├─────────────────┼──────────────────────┼──────────────────────┤
│ Lock 순서 분석  │ ✓ (강점)             │ 제한적               │
├─────────────────┼──────────────────────┼──────────────────────┤
│ 재컴파일 필요   │ 아니오               │ 예                   │
└─────────────────┴──────────────────────┴──────────────────────┘
```

## Detection Strategies

### 1. Code Review Checklist

Look for these patterns:
- [ ] Unsynchronized access to shared variables
- [ ] Check-then-act sequences
- [ ] Read-modify-write operations
- [ ] Multiple locks held simultaneously
- [ ] Lazy initialization without proper synchronization

### 2. Static Analysis

```bash
# Using Clang Thread Safety Analysis
clang -Wthread-safety -c program.c

# Using Coverity (commercial)
cov-analyze --dir output --enable-constraint-fpp
```

### 3. Dynamic Detection with ThreadSanitizer

```bash
# Compile with TSan
gcc -fsanitize=thread -g -O1 race_condition.c -o race_condition -lpthread

# Run the program
./race_condition

# Sample output:
# ==================
# WARNING: ThreadSanitizer: data race (pid=12345)
#   Write of size 4 at 0x7b0400001000 by thread T2:
#     #0 increment race_condition.c:8
#   Previous write of size 4 at 0x7b0400001000 by thread T1:
#     #0 increment race_condition.c:8
```

### 4. Stress Testing

```c
#include <pthread.h>
#include <stdio.h>

#define NUM_THREADS 100
#define ITERATIONS 10000

// Your potentially racy code here

int main() {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, test_function, NULL);
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // Verify results
    // If results vary across runs, you have a race!
    return 0;
}
```

## Prevention Best Practices

### 1. Minimize Shared State

```c
// GOOD: Thread-local storage
__thread int thread_counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        thread_counter++;  // No synchronization needed
    }
    return NULL;
}
```

### 2. Immutable Data Structures

```c
// GOOD: Immutable design
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// Instead of modifying, create new nodes
Node* prepend(Node* head, int value) {
    Node* new_node = malloc(sizeof(Node));
    new_node->value = value;
    new_node->next = head;
    return new_node;  // Return new head
}
```

### 3. Atomic Operations for Simple Cases

```c
#include <stdatomic.h>

atomic_int counter = 0;

// Simple increment - no mutex needed
atomic_fetch_add(&counter, 1);

// Compare-and-swap for more complex operations
int expected = 5;
int desired = 10;
atomic_compare_exchange_strong(&counter, &expected, desired);
```

### 4. Lock Granularity

```c
// BAD: Coarse-grained locking
pthread_mutex_t global_lock;

void operation1() {
    pthread_mutex_lock(&global_lock);
    // ... lots of work ...
    pthread_mutex_unlock(&global_lock);
}

// GOOD: Fine-grained locking
typedef struct {
    int data;
    pthread_mutex_t mutex;
} DataItem;

DataItem items[100];

void operation2(int index) {
    pthread_mutex_lock(&items[index].mutex);
    // ... work on specific item ...
    pthread_mutex_unlock(&items[index].mutex);
}
```

### 5. Lock-Free Data Structures

```c
// Lock-free stack using CAS
#include <stdatomic.h>
#include <stdlib.h>

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
```

## Performance Considerations

### Cost of Different Approaches

```
┌────────────────────────┬──────────────┬────────────────┐
│       Technique        │   Overhead   │   Scalability  │
├────────────────────────┼──────────────┼────────────────┤
│ No Synchronization     │     None     │   Excellent    │
│ Atomic Operations      │     Low      │      Good      │
│ Mutex (uncontended)    │    Medium    │      Good      │
│ Mutex (contended)      │     High     │      Poor      │
│ Global Lock            │   Very High  │   Very Poor    │
└────────────────────────┴──────────────┴────────────────┘
```

## Common Pitfalls

### Pitfall 1: Assuming Single Operations Are Atomic

```c
// WRONG: Even simple assignments can be non-atomic
long long value = 0;  // 64-bit on 32-bit system

// Thread 1
value = 0x0000000100000001;  // Might be two 32-bit writes

// Thread 2
long long temp = value;  // Might read: 0x0000000000000001
```

### Pitfall 2: Volatile Does Not Mean Thread-Safe

```c
// WRONG: volatile does NOT provide synchronization
volatile int flag = 0;

// Thread 1
data = 42;
flag = 1;  // Signal to thread 2

// Thread 2
while (flag == 0);  // Wait for signal
use(data);  // NOT GUARANTEED to see data = 42!

// CORRECT: Use atomic or mutex
```

### Pitfall 3: Compiler/CPU Reordering

```c
// Can be reordered by compiler or CPU
int data = 0;
int ready = 0;

// Thread 1
data = 42;
ready = 1;  // Can execute before data = 42!

// Thread 2
while (ready == 0);
use(data);  // Might not see 42!

// SOLUTION: Use atomic with proper memory ordering
atomic_store_explicit(&ready, 1, memory_order_release);
while (atomic_load_explicit(&ready, memory_order_acquire) == 0);
```

## Exercises

### Exercise 1: Fix the Bug
```c
// Find and fix the race condition
#include <pthread.h>

int balance = 1000;

void* transfer(void* arg) {
    int amount = *(int*)arg;
    int temp = balance;
    temp -= amount;
    balance = temp;
    return NULL;
}
```

### Exercise 2: Thread-Safe Queue
Implement a thread-safe FIFO queue with `enqueue` and `dequeue` operations.

### Exercise 3: Concurrent Hash Table
Implement a hash table that supports concurrent reads and writes with fine-grained locking.

## Summary

Race conditions are:
- **Insidious**: Hard to detect and reproduce
- **Common**: Appear in most concurrent programs
- **Fixable**: With proper synchronization
- **Preventable**: With careful design

### Key Takeaways

1. **Always synchronize** shared mutable state
2. **Use atomic operations** for simple counters and flags
3. **Test with ThreadSanitizer** to catch races
4. **Design for immutability** when possible
5. **Minimize critical sections** for performance

## Further Reading

- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Is Parallel Programming Hard?" - Paul McKenney
- ThreadSanitizer documentation
- C11 Atomic operations reference

## Next Topic

Continue to [02-deadlock.md](./02-deadlock.md) to learn about deadlocks.
