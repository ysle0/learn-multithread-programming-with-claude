# Race Condition (경쟁 조건)

## Race Condition이란?

**Race Condition(경쟁 조건)**은 프로그램의 동작이 여러 스레드의 상대적인 타이밍이나 인터리빙에 의존할 때 발생합니다. 둘 이상의 스레드가 공유 데이터에 동시에 접근하고, 최소 하나 이상의 스레드가 데이터를 수정할 때, 최종 결과는 예측 불가능해지며 어느 스레드가 "경쟁에서 이기는지"에 따라 달라집니다.

### 형식적 정의

다음 조건이 모두 만족될 때 Race Condition이 존재합니다:
1. 둘 이상의 스레드가 동일한 메모리 위치에 접근
2. 최소 하나의 접근이 쓰기 연산
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
                                                counter = 1 (잘못됨!)

기대값: counter = 2
실제값: counter = 1
```

### 타임라인 다이어그램

```
Time ─────────────────────────────────────────────────────▶

Thread 1: [─R─][─+─][─W─]      [─R─][─+─][─W─]
Thread 2:      [─R─][─+─][─W─]      [─R─][─+─][─W─]

범례: R=Read(읽기), +=Compute(계산), W=Write(쓰기)

중첩된 연산이 Race Condition을 유발!
```

## Race Condition의 유형

### 1. Read-Modify-Write Race

가장 일반적인 유형으로, 여러 스레드가 값을 읽고, 수정하고, 다시 쓰는 경우입니다.

```c
// 문제: counter에 대한 Race Condition
#include <pthread.h>
#include <stdio.h>

int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // 원자적이지 않음: read, increment, write
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Counter: %d (기대값: 2000000)\n", counter);
    // 출력 결과 다양: 1000000, 1500000, 1850000, 등
    return 0;
}
```

**왜 발생하는가:**
```assembly
; counter++는 여러 명령어로 컴파일됨:
MOV  eax, [counter]   ; 현재 값 읽기
INC  eax              ; 레지스터에서 증가
MOV  [counter], eax   ; 메모리에 다시 쓰기

; 스레드 인터리빙은 이들 사이 어디서든 발생 가능!
```

**해결책 1: Mutex 사용**
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

    printf("Counter: %d (기대값: 2000000)\n", counter);
    // 출력: 항상 2000000

    pthread_mutex_destroy(&mutex);
    return 0;
}
```

**해결책 2: Atomic 연산 사용**
```c
#include <pthread.h>
#include <stdatomic.h>
#include <stdio.h>

atomic_int counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        atomic_fetch_add(&counter, 1);  // 원자적 연산
    }
    return NULL;
}

int main() {
    pthread_t t1, t2;

    pthread_create(&t1, NULL, increment, NULL);
    pthread_create(&t2, NULL, increment, NULL);

    pthread_join(t1, NULL);
    pthread_join(t2, NULL);

    printf("Counter: %d (기대값: 2000000)\n", counter);
    // 출력: 항상 2000000
    return 0;
}
```

### 2. Check-Then-Act Race

스레드가 조건을 확인하고 그 결과에 따라 행동하지만, 확인과 행동 사이에 조건이 변경될 수 있는 경우입니다.

```c
// 문제: Check-then-act Race Condition
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int balance;
} BankAccount;

BankAccount account = {1000};

void* withdraw(void* arg) {
    int amount = *(int*)arg;

    // 확인 (CHECK)
    if (account.balance >= amount) {
        // 여기서 컨텍스트 스위치 발생 가능!
        // 행동 (ACT)
        account.balance -= amount;
        printf("출금 %d원, 잔액: %d원\n", amount, account.balance);
    } else {
        printf("잔액 부족\n");
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

    printf("최종 잔액: %d원 (기대값: >= 0)\n", account.balance);
    // 출력 가능: 최종 잔액: -200원 (초과 인출!)
    return 0;
}
```

**버그 타임라인:**
```
초기 잔액: 1000원

Thread 1                    Thread 2                    Balance
--------                    --------                    -------
확인: balance >= 600 ✓
                            확인: balance >= 600 ✓
출금 600
                            출금 600
                                                        -200 (버그!)
```

**해결책: Atomic Check-and-Act**
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
        printf("출금 %d원, 잔액: %d원\n", amount, account.balance);
    } else {
        printf("잔액 부족\n");
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

    printf("최종 잔액: %d원\n", account.balance);
    // 출력: 항상 >= 0

    pthread_mutex_destroy(&account.mutex);
    return 0;
}
```

### 3. Lazy Initialization Race (Double-Checked Locking)

싱글톤 패턴에서 발생하는 미묘한 Race Condition입니다.

```c
// 문제: 잘못된 Double-Checked Locking
#include <pthread.h>
#include <stdlib.h>
#include <stdio.h>

typedef struct {
    int data;
} Singleton;

Singleton* instance = NULL;
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

Singleton* get_instance() {
    if (instance == NULL) {  // 첫 번째 확인 (동기화 안됨)
        pthread_mutex_lock(&mutex);
        if (instance == NULL) {  // 두 번째 확인
            instance = malloc(sizeof(Singleton));
            instance->data = 42;  // 초기화
        }
        pthread_mutex_unlock(&mutex);
    }
    return instance;
}

// 버그: 컴파일러/CPU가 재배치 가능:
// 1. 메모리 할당
// 2. instance에 할당
// 3. data 초기화
// Thread 2가 NULL이 아니지만 초기화되지 않은 instance를 볼 수 있음!
```

**해결책: Atomic 연산 사용**
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

### 예제 1: 스레드 안전 스택

```c
// 문제: 스택 연산의 Race Condition
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

// 여러 스레드가 push/pop을 호출하면 혼란 발생!
```

**해결책: Lock 기반 스레드 안전 스택**
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

### 예제 2: Reference Counting

```c
// 문제: Reference Counting의 Race Condition
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

**해결책: Atomic Reference Counting**
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

## 탐지 전략

### 1. 코드 리뷰 체크리스트

다음 패턴을 찾으세요:
- [ ] 공유 변수에 대한 동기화되지 않은 접근
- [ ] Check-then-act 시퀀스
- [ ] Read-modify-write 연산
- [ ] 동시에 보유한 여러 lock
- [ ] 적절한 동기화 없는 Lazy Initialization

### 2. 정적 분석

```bash
# Clang Thread Safety Analysis 사용
clang -Wthread-safety -c program.c

# Coverity 사용 (상용)
cov-analyze --dir output --enable-constraint-fpp
```

### 3. ThreadSanitizer를 사용한 동적 탐지

```bash
# TSan으로 컴파일
gcc -fsanitize=thread -g -O1 race_condition.c -o race_condition -lpthread

# 프로그램 실행
./race_condition

# 샘플 출력:
# ==================
# WARNING: ThreadSanitizer: data race (pid=12345)
#   Write of size 4 at 0x7b0400001000 by thread T2:
#     #0 increment race_condition.c:8
#   Previous write of size 4 at 0x7b0400001000 by thread T1:
#     #0 increment race_condition.c:8
```

### 4. 스트레스 테스팅

```c
#include <pthread.h>
#include <stdio.h>

#define NUM_THREADS 100
#define ITERATIONS 10000

// Race 가능성이 있는 코드 여기에

int main() {
    pthread_t threads[NUM_THREADS];

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, test_function, NULL);
    }

    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    // 결과 검증
    // 실행마다 결과가 다르면 Race가 있음!
    return 0;
}
```

## 예방 모범 사례

### 1. 공유 상태 최소화

```c
// 좋음: Thread-Local Storage
__thread int thread_counter = 0;

void* increment(void* arg) {
    for (int i = 0; i < 1000000; i++) {
        thread_counter++;  // 동기화 불필요
    }
    return NULL;
}
```

### 2. 불변 데이터 구조

```c
// 좋음: 불변 설계
typedef struct Node {
    int value;
    struct Node* next;
} Node;

// 수정하는 대신 새 노드 생성
Node* prepend(Node* head, int value) {
    Node* new_node = malloc(sizeof(Node));
    new_node->value = value;
    new_node->next = head;
    return new_node;  // 새 head 반환
}
```

### 3. 간단한 경우 Atomic 연산

```c
#include <stdatomic.h>

atomic_int counter = 0;

// 간단한 증가 - mutex 불필요
atomic_fetch_add(&counter, 1);

// 더 복잡한 연산을 위한 Compare-and-swap
int expected = 5;
int desired = 10;
atomic_compare_exchange_strong(&counter, &expected, desired);
```

### 4. Lock 세분성 (Lock Granularity)

```c
// 나쁨: 거친 세분성 locking
pthread_mutex_t global_lock;

void operation1() {
    pthread_mutex_lock(&global_lock);
    // ... 많은 작업 ...
    pthread_mutex_unlock(&global_lock);
}

// 좋음: 세밀한 세분성 locking
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

### 5. Lock-Free 자료구조

```c
// CAS를 사용한 Lock-Free 스택
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

## 성능 고려사항

### 다양한 접근법의 비용

```
┌────────────────────────┬──────────────┬────────────────┐
│       기법             │   오버헤드   │   확장성       │
├────────────────────────┼──────────────┼────────────────┤
│ 동기화 없음            │     없음     │   우수         │
│ Atomic 연산            │     낮음     │   양호         │
│ Mutex (경합 없음)      │    중간      │   양호         │
│ Mutex (경합 있음)      │     높음     │   나쁨         │
│ 전역 Lock              │   매우 높음  │   매우 나쁨    │
└────────────────────────┴──────────────┴────────────────┘
```

## 흔한 함정

### 함정 1: 단일 연산이 원자적이라고 가정

```c
// 잘못됨: 단순 할당도 비원자적일 수 있음
long long value = 0;  // 32비트 시스템에서 64비트

// Thread 1
value = 0x0000000100000001;  // 두 번의 32비트 쓰기일 수 있음

// Thread 2
long long temp = value;  // 읽을 수 있음: 0x0000000000000001
```

### 함정 2: Volatile이 스레드 안전을 의미하지 않음

```c
// 잘못됨: volatile은 동기화를 제공하지 않음
volatile int flag = 0;

// Thread 1
data = 42;
flag = 1;  // Thread 2에게 신호

// Thread 2
while (flag == 0);  // 신호 대기
use(data);  // data = 42를 보장받지 못함!

// 올바름: Atomic 또는 Mutex 사용
```

### 함정 3: 컴파일러/CPU 재배치

```c
// 컴파일러나 CPU가 재배치 가능
int data = 0;
int ready = 0;

// Thread 1
data = 42;
ready = 1;  // data = 42 이전에 실행될 수 있음!

// Thread 2
while (ready == 0);
use(data);  // 42를 보지 못할 수 있음!

// 해결책: 적절한 메모리 순서로 Atomic 사용
atomic_store_explicit(&ready, 1, memory_order_release);
while (atomic_load_explicit(&ready, memory_order_acquire) == 0);
```

## 연습 문제

### 연습 1: 버그 수정
```c
// Race Condition을 찾아 수정하세요
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

### 연습 2: 스레드 안전 큐
`enqueue`와 `dequeue` 연산을 가진 스레드 안전 FIFO 큐를 구현하세요.

### 연습 3: 동시성 해시 테이블
세밀한 세분성 locking으로 동시 읽기와 쓰기를 지원하는 해시 테이블을 구현하세요.

## 요약

Race Condition은:
- **은밀함**: 탐지 및 재현이 어려움
- **흔함**: 대부분의 동시성 프로그램에서 나타남
- **수정 가능**: 적절한 동기화로 해결
- **예방 가능**: 신중한 설계로 방지

### 핵심 포인트

1. **항상 동기화**: 공유 가변 상태
2. **Atomic 연산 사용**: 간단한 카운터와 플래그에
3. **ThreadSanitizer로 테스트**: Race 탐지
4. **불변성 설계**: 가능한 경우
5. **임계 영역 최소화**: 성능을 위해

## 추가 자료

- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Is Parallel Programming Hard?" - Paul McKenney
- ThreadSanitizer 문서
- C11 Atomic 연산 레퍼런스

## 다음 주제

[02-deadlock.md](./02-deadlock.md)에서 Deadlock에 대해 학습하세요.
