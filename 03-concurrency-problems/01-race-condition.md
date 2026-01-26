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

```cpp
// 문제: counter에 대한 Race Condition
#include <thread>
#include <iostream>

int counter = 0;

void increment() {
    for (int i = 0; i < 1000000; i++) {
        counter++;  // 원자적이지 않음: read, increment, write
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << " (기대값: 2000000)\n";
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
```cpp
#include <thread>
#include <mutex>
#include <iostream>

int counter = 0;
std::mutex mutex;

void increment() {
    for (int i = 0; i < 1000000; i++) {
        std::lock_guard<std::mutex> lock(mutex);
        counter++;
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << " (기대값: 2000000)\n";
    // 출력: 항상 2000000

    return 0;
}
```

**해결책 2: Atomic 연산 사용**
```cpp
#include <thread>
#include <atomic>
#include <iostream>

std::atomic<int> counter = 0;

void increment() {
    for (int i = 0; i < 1000000; i++) {
        counter.fetch_add(1);  // 원자적 연산
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << counter << " (기대값: 2000000)\n";
    // 출력: 항상 2000000
    return 0;
}
```

### 2. Check-Then-Act Race

스레드가 조건을 확인하고 그 결과에 따라 행동하지만, 확인과 행동 사이에 조건이 변경될 수 있는 경우입니다.

```cpp
// 문제: Check-then-act Race Condition
#include <thread>
#include <iostream>

struct BankAccount {
    int balance;
};

BankAccount account = {1000};

void withdraw(int amount) {
    // 확인 (CHECK)
    if (account.balance >= amount) {
        // 여기서 컨텍스트 스위치 발생 가능!
        // 행동 (ACT)
        account.balance -= amount;
        std::cout << "출금 " << amount << "원, 잔액: " << account.balance << "원\n";
    } else {
        std::cout << "잔액 부족\n";
    }
}

int main() {
    int amount1 = 600, amount2 = 600;

    std::thread t1(withdraw, amount1);
    std::thread t2(withdraw, amount2);

    t1.join();
    t2.join();

    std::cout << "최종 잔액: " << account.balance << "원 (기대값: >= 0)\n";
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
```cpp
#include <thread>
#include <mutex>
#include <iostream>

struct BankAccount {
    int balance;
    std::mutex mutex;
};

BankAccount account = {1000};

bool withdraw(int amount) {
    std::lock_guard<std::mutex> lock(account.mutex);

    bool success = false;
    if (account.balance >= amount) {
        account.balance -= amount;
        success = true;
        std::cout << "출금 " << amount << "원, 잔액: " << account.balance << "원\n";
    } else {
        std::cout << "잔액 부족\n";
    }

    return success;
}

int main() {
    int amount1 = 600, amount2 = 600;

    std::thread t1(withdraw, amount1);
    std::thread t2(withdraw, amount2);

    t1.join();
    t2.join();

    std::cout << "최종 잔액: " << account.balance << "원\n";
    // 출력: 항상 >= 0

    return 0;
}
```

### 3. Lazy Initialization Race (Double-Checked Locking)

싱글톤 패턴에서 발생하는 미묘한 Race Condition입니다.

```cpp
// 문제: 잘못된 Double-Checked Locking
#include <mutex>
#include <iostream>

struct Singleton {
    int data;
};

Singleton* instance = nullptr;
std::mutex mutex;

Singleton* get_instance() {
    if (instance == nullptr) {  // 첫 번째 확인 (동기화 안됨)
        std::lock_guard<std::mutex> lock(mutex);
        if (instance == nullptr) {  // 두 번째 확인
            instance = new Singleton();
            instance->data = 42;  // 초기화
        }
    }
    return instance;
}

// 버그: 컴파일러/CPU가 재배치 가능:
// 1. 메모리 할당
// 2. instance에 할당
// 3. data 초기화
// Thread 2가 nullptr이 아니지만 초기화되지 않은 instance를 볼 수 있음!
```

**해결책: Atomic 연산 사용**
```cpp
#include <mutex>
#include <atomic>

struct Singleton {
    int data;
};

std::atomic<Singleton*> instance{nullptr};
std::mutex mutex;

Singleton* get_instance() {
    Singleton* tmp = instance.load();

    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(mutex);
        tmp = instance.load();
        if (tmp == nullptr) {
            tmp = new Singleton();
            tmp->data = 42;
            instance.store(tmp);
        }
    }

    return tmp;
}
```

## 실전 예제

### 예제 1: 스레드 안전 스택

```cpp
// 문제: 스택 연산의 Race Condition
#include <array>

constexpr int MAX_SIZE = 100;

struct Stack {
    std::array<int, MAX_SIZE> items;
    int top;
};

Stack stack = {{}, -1};

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
```cpp
#include <mutex>
#include <array>

constexpr int MAX_SIZE = 100;

class ThreadSafeStack {
private:
    std::array<int, MAX_SIZE> items;
    int top;
    std::mutex mutex;

public:
    ThreadSafeStack() : top(-1) {}

    bool push(int value) {
        std::lock_guard<std::mutex> lock(mutex);

        if (top >= MAX_SIZE - 1) {
            return false;
        }

        top++;
        items[top] = value;
        return true;
    }

    bool pop(int* value) {
        std::lock_guard<std::mutex> lock(mutex);

        if (top < 0) {
            return false;
        }

        *value = items[top];
        top--;
        return true;
    }
};
```

### 예제 2: Reference Counting

```cpp
// 문제: Reference Counting의 Race Condition
struct SharedObject {
    int* data;
    int ref_count;
};

void acquire(SharedObject* obj) {
    obj->ref_count++;  // RACE CONDITION!
}

void release(SharedObject* obj) {
    obj->ref_count--;  // RACE CONDITION!
    if (obj->ref_count == 0) {
        delete obj->data;
        delete obj;
    }
}
```

**해결책: Atomic Reference Counting**
```cpp
#include <atomic>

struct SharedObject {
    int* data;
    std::atomic<int> ref_count;
};

void acquire(SharedObject* obj) {
    obj->ref_count.fetch_add(1);
}

void release(SharedObject* obj) {
    if (obj->ref_count.fetch_sub(1) == 1) {
        // 마지막 참조였음
        delete obj->data;
        delete obj;
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

```cpp
#include <thread>
#include <vector>
#include <iostream>

constexpr int NUM_THREADS = 100;
constexpr int ITERATIONS = 10000;

// Race 가능성이 있는 코드 여기에

int main() {
    std::vector<std::thread> threads;

    for (int i = 0; i < NUM_THREADS; i++) {
        threads.emplace_back(test_function);
    }

    for (auto& t : threads) {
        t.join();
    }

    // 결과 검증
    // 실행마다 결과가 다르면 Race가 있음!
    return 0;
}
```

## 예방 모범 사례

### 1. 공유 상태 최소화

```cpp
// 좋음: Thread-Local Storage
thread_local int thread_counter = 0;

void increment() {
    for (int i = 0; i < 1000000; i++) {
        thread_counter++;  // 동기화 불필요
    }
}
```

### 2. 불변 데이터 구조

```cpp
// 좋음: 불변 설계
struct Node {
    int value;
    Node* next;
};

// 수정하는 대신 새 노드 생성
Node* prepend(Node* head, int value) {
    Node* new_node = new Node();
    new_node->value = value;
    new_node->next = head;
    return new_node;  // 새 head 반환
}
```

### 3. 간단한 경우 Atomic 연산

```cpp
#include <atomic>

std::atomic<int> counter = 0;

// 간단한 증가 - mutex 불필요
counter.fetch_add(1);

// 더 복잡한 연산을 위한 Compare-and-swap
int expected = 5;
int desired = 10;
counter.compare_exchange_strong(expected, desired);
```

### 4. Lock 세분성 (Lock Granularity)

```cpp
// 나쁨: 거친 세분성 locking
std::mutex global_lock;

void operation1() {
    std::lock_guard<std::mutex> lock(global_lock);
    // ... 많은 작업 ...
}

// 좋음: 세밀한 세분성 locking
struct DataItem {
    int data;
    std::mutex mutex;
};

DataItem items[100];

void operation2(int index) {
    std::lock_guard<std::mutex> lock(items[index].mutex);
    // ... 특정 항목에 대한 작업 ...
}
```

### 5. Lock-Free 자료구조

```cpp
// CAS를 사용한 Lock-Free 스택
#include <atomic>

struct Node {
    int value;
    Node* next;
};

class LockFreeStack {
private:
    std::atomic<Node*> head;

public:
    LockFreeStack() : head(nullptr) {}

    void push(int value) {
        Node* new_node = new Node();
        new_node->value = value;

        Node* old_head = head.load();
        do {
            new_node->next = old_head;
        } while (!head.compare_exchange_weak(old_head, new_node));
    }
};
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

```cpp
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

```cpp
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
std::atomic<int> ready_atomic{0};
ready_atomic.store(1, std::memory_order_release);
while (ready_atomic.load(std::memory_order_acquire) == 0);
```

## 연습 문제

### 연습 1: 버그 수정
```cpp
// Race Condition을 찾아 수정하세요
#include <thread>

int balance = 1000;

void transfer(int amount) {
    int temp = balance;
    temp -= amount;
    balance = temp;
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
