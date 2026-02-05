# Race Conditions

## Race Condition이란?

**Race condition**은 프로그램의 동작이 여러 thread의 상대적인 타이밍이나 인터리빙에 따라 달라질 때 발생합니다. 두 개 이상의 thread가 공유 데이터에 동시에 접근하고, 그 중 하나 이상의 thread가 데이터를 수정할 때, 최종 결과는 예측 불가능해지며 어떤 thread가 "경쟁에서 이기느냐"에 따라 달라집니다.

### 형식적 정의

Race condition은 다음 조건들이 충족될 때 존재합니다:
1. 두 개 이상의 thread가 같은 메모리 위치에 접근
2. 하나 이상의 접근이 쓰기 연산
3. 접근이 동기화되지 않음
4. 결과가 실행 타이밍에 의존

## 시각적 표현

### 비결정적 실행

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

### 타임라인 다이어그램

```
Time ─────────────────────────────────────────────────────▶

Thread 1: [─R─][─+─][─W─]      [─R─][─+─][─W─]
Thread 2:      [─R─][─+─][─W─]      [─R─][─+─][─W─]

Legend: R=Read, +=Compute, W=Write

Overlapping operations cause race condition!
```

## Race Condition의 유형

### 1. Read-Modify-Write Race

가장 흔한 유형으로, 여러 thread가 값을 읽고, 수정하고, 다시 쓰는 경우입니다.

```c
// 문제: counter에 대한 race condition
#include <pthread.h>
#include <stdio.h>

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // ATOMIC이 아님: 읽기, 증가, 쓰기
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
    // 출력이 매번 다름: 1000000, 1500000, 1850000 등
    return 0;
}
```

**왜 발생하는가:**
```assembly
; counter++는 여러 명령어로 컴파일됨:
MOV  eax, [counter]   ; 현재 값 읽기
INC  eax              ; 레지스터에서 증가
MOV  [counter], eax   ; 메모리에 다시 쓰기

; 이 명령어들 사이에서 thread 인터리빙이 발생할 수 있음!
```

**해결 방법 1: Mutex 사용**
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
    // 출력: 항상 2000000

    pthread_mutex_destroy(&mutex);
    return 0;
}
```

**해결 방법 2: Atomic 연산 사용**
```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

atomic_int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1);  // Atomic 연산
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
    // 출력: 항상 2000000
    return 0;
}
```

### 2. Check-Then-Act Race

Thread가 조건을 확인한 후 그 결과에 따라 행동하지만, 확인과 행동 사이에 조건이 바뀔 수 있는 경우입니다.

```c
// 문제: Check-then-act race condition
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int balance;
} BankAccount;

BankAccount account = {1000};

void* withdraw(void* arg) {
    int amount = *(int*)arg;

    // 확인
    if (account.balance >= amount) {
        // 여기서 컨텍스트 스위치가 발생할 수 있음!
        // 행동
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
    // 출력 가능: Final balance: -200 (초과 인출!)
    return 0;
}
```

**버그의 타임라인:**
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

**해결 방법: Atomic Check-and-Act**
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
    // 출력: 항상 >= 0

    pthread_mutex_destroy(&account.mutex);
    return 0;
}
```

### 3. 지연 초기화 Race (Double-Checked Locking)

싱글턴 패턴에서 발생하는 미묘한 race condition입니다.

```c
// 문제: 깨진 double-checked locking
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    int data;
} Singleton;

Singleton* instance = NULL;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

Singleton* get_instance() {
    if (instance == NULL) {  // 첫 번째 확인 (동기화 안 됨)
        pthread_mutex_lock(&mutex);
        if (instance == NULL) {  // 두 번째 확인
            instance = malloc(sizeof(Singleton));
            instance->data = 42;  // 초기화
        }
        pthread_mutex_unlock(&mutex);
    }
    return instance;
}

// 버그: 컴파일러/CPU가 순서를 변경할 수 있음:
// 1. 메모리 할당
// 2. instance에 할당
// 3. data 초기화
// Thread 2가 NULL이 아닌 초기화되지 않은 instance를 볼 수 있음!
```

**해결 방법: Atomic 연산 사용**
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

## 실전 예제

### 예제 1: Thread 안전 스택

```c
// 문제: 스택 연산에서의 race condition
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

// 여러 thread가 push/pop을 호출하면 혼란이 발생!
```

**해결 방법: Lock 기반 Thread 안전 스택**
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

### 예제 2: 참조 카운팅

```c
// 문제: 참조 카운팅에서의 race condition
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

**해결 방법: Atomic 참조 카운팅**
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
        // 마지막 참조였음
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

## 탐지 전략

### 1. 코드 리뷰 체크리스트

다음 패턴들을 확인하세요:
- [ ] 동기화되지 않은 공유 변수 접근
- [ ] Check-then-act 시퀀스
- [ ] Read-modify-write 연산
- [ ] 동시에 여러 lock을 보유하는 경우
- [ ] 적절한 동기화 없는 지연 초기화

### 2. 정적 분석

```bash
# Clang Thread Safety Analysis 사용
clang -Wthread-safety -c program.c

# Coverity 사용 (상용)
cov-analyze --dir output --enable-constraint-fpp
```

### 3. ThreadSanitizer를 이용한 동적 탐지

```bash
# TSan으로 컴파일
gcc -fsanitize=thread -g -O1 race_condition.c -o race_condition -lpthread

# 프로그램 실행
./race_condition

# 출력 예시:
# ==================
# WARNING: ThreadSanitizer: data race (pid=12345)
#   Write of size 4 at 0x7b0400001000 by thread T2:
#     #0 increment race_condition.c:8
#   Previous write of size 4 at 0x7b0400001000 by thread T1:
#     #0 increment race_condition.c:8
```

### 4. 스트레스 테스트

```c
#include <pthread.h>
#include <stdio.h>

#define NUM_THREADS 100
#define ITERATIONS 10000

// 잠재적으로 race가 있는 코드를 여기에 작성

int main() {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, test_function, NULL);
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // 결과 검증
    // 실행할 때마다 결과가 달라진다면 race가 있는 것!
    return 0;
}
```

## 예방 모범 사례

### 1. 공유 상태 최소화

```c
// 좋은 예: Thread-local storage
__thread int thread_counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        thread_counter++;  // 동기화 필요 없음
    }
    return NULL;
}
```

### 2. 불변 데이터 구조

```c
// 좋은 예: 불변 설계
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// 수정하는 대신 새 노드를 생성
Node* prepend(Node* head, int value) {
    Node* new_node = malloc(sizeof(Node));
    new_node->value = value;
    new_node->next = head;
    return new_node;  // 새로운 head 반환
}
```

### 3. 단순한 경우에 Atomic 연산 사용

```c
#include <stdatomic.h>

atomic_int counter = 0;

// 단순 증가 - mutex 불필요
atomic_fetch_add(&counter, 1);

// 더 복잡한 연산에는 compare-and-swap 사용
int expected = 5;
int desired = 10;
atomic_compare_exchange_strong(&counter, &expected, desired);
```

### 4. Lock 세분화

```c
// 나쁜 예: 거친 세분화 locking
pthread_mutex_t global_lock;

void operation1() {
    pthread_mutex_lock(&global_lock);
    // ... 많은 작업 ...
    pthread_mutex_unlock(&global_lock);
}

// 좋은 예: 세밀한 세분화 locking
typedef struct {
    int data;
    pthread_mutex_t mutex;
} DataItem;

DataItem items[100];

void operation2(int index) {
    pthread_mutex_lock(&items[index].mutex);
    // ... 특정 항목에 대한 작업 ...
    pthread_mutex_unlock(&items[index].mutex);
}
```

### 5. Lock-Free 데이터 구조

```c
// CAS를 사용한 lock-free 스택
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

## 성능 고려 사항

### 접근 방식별 비용

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

## 흔한 실수

### 실수 1: 단일 연산이 atomic이라고 가정하기

```c
// 잘못된 예: 단순 대입도 atomic이 아닐 수 있음
long long value = 0;  // 32비트 시스템에서 64비트

// Thread 1
value = 0x0000000100000001;  // 두 번의 32비트 쓰기일 수 있음

// Thread 2
long long temp = value;  // 읽은 값: 0x0000000000000001일 수 있음
```

### 실수 2: volatile이 Thread 안전을 의미하지 않음

```c
// 잘못된 예: volatile은 동기화를 제공하지 않음
volatile int flag = 0;

// Thread 1
data = 42;
flag = 1;  // Thread 2에 신호 보내기

// Thread 2
while (flag == 0);  // 신호 대기
use(data);  // data = 42를 볼 수 있다는 보장이 없음!

// 올바른 방법: atomic 또는 mutex 사용
```

### 실수 3: 컴파일러/CPU 재배치

```c
// 컴파일러 또는 CPU에 의해 재배치될 수 있음
int data = 0;
int ready = 0;

// Thread 1
data = 42;
ready = 1;  // data = 42보다 먼저 실행될 수 있음!

// Thread 2
while (ready == 0);
use(data);  // 42를 못 볼 수 있음!

// 해결 방법: 적절한 메모리 순서와 함께 atomic 사용
atomic_store_explicit(&ready, 1, memory_order_release);
while (atomic_load_explicit(&ready, memory_order_acquire) == 0);
```

## 연습 문제

### 연습 문제 1: 버그 수정
```c
// Race condition을 찾아서 수정하세요
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

### 연습 문제 2: Thread 안전 큐
`enqueue`와 `dequeue` 연산이 있는 thread 안전 FIFO 큐를 구현하세요.

### 연습 문제 3: 동시성 해시 테이블
세밀한 세분화 locking을 사용하여 동시 읽기와 쓰기를 지원하는 해시 테이블을 구현하세요.

## 요약

Race condition은:
- **교활함**: 탐지와 재현이 어려움
- **흔함**: 대부분의 동시성 프로그램에서 나타남
- **수정 가능**: 적절한 동기화로 해결 가능
- **예방 가능**: 신중한 설계로 방지 가능

### 핵심 요점

1. 공유 가변 상태는 **항상 동기화**할 것
2. 단순한 카운터와 플래그에는 **atomic 연산 사용**
3. **ThreadSanitizer로 테스트**하여 race를 잡을 것
4. 가능하면 **불변성을 고려한 설계**
5. 성능을 위해 **임계 구역 최소화**

## 추가 참고 자료

- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Is Parallel Programming Hard?" - Paul McKenney
- ThreadSanitizer 문서
- C11 Atomic 연산 레퍼런스

## 다음 주제

[02-deadlock.md](./02-deadlock.md)로 이동하여 deadlock에 대해 알아보세요.
