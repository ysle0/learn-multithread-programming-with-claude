# ABA Problem

## 목차
1. [개요](#개요)
2. [ABA 문제란?](#aba-문제란)
3. [실제 발생 시나리오](#실제-발생-시나리오)
4. [해결책](#해결책)
5. [구현 예제](#구현-예제)
6. [언어별 지원](#언어별-지원)
7. [요약](#요약)

---

## 개요

**ABA Problem**은 Lock-Free 알고리즘에서 CAS(Compare-And-Swap) 연산을 사용할 때 발생할 수 있는 미묘하지만 치명적인 버그입니다. 값이 A에서 B로, 다시 A로 변경되었을 때 CAS가 이를 감지하지 못하는 문제입니다.

### 핵심 문제

```
┌─────────────────────────────────────────────────────────────────┐
│                      ABA Problem 요약                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CAS의 한계:                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CAS는 "값"만 비교한다                                     │   │
│  │ → 값이 같으면 "변하지 않았다"고 판단                       │   │
│  │ → 하지만 A → B → A 로 변했다면?                          │   │
│  │    값은 같지만 상태는 완전히 달라졌을 수 있음!             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  비유:                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 은행 계좌 잔액 = $100                                     │   │
│  │                                                          │   │
│  │ T1: "잔액이 $100이면 $50 출금"                           │   │
│  │ T1: 잔액 확인 → $100 ✓                                   │   │
│  │ T1: (잠깐 멈춤)                                          │   │
│  │                                                          │   │
│  │ T2: $100 출금 → 잔액 $0                                  │   │
│  │ T3: $100 입금 → 잔액 $100                                │   │
│  │                                                          │   │
│  │ T1: CAS($100 → $50) → 성공! (잔액이 $100이므로)          │   │
│  │                                                          │   │
│  │ 문제: T1은 중간 거래를 모름, 의도치 않은 출금 발생        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## ABA 문제란?

### 기본 시나리오

```
┌─────────────────────────────────────────────────────────────────┐
│                    ABA 문제 타임라인                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  시간   Thread 1              Thread 2              메모리      │
│  ────────────────────────────────────────────────────────────   │
│  t0     read(ptr) = A                               ptr → A    │
│         (A의 next = B)                                          │
│                                                                 │
│  t1     (preempted)           ────────────────                  │
│                                                                 │
│  t2                           pop() → A            ptr → B     │
│                               (A 해제됨)                        │
│                                                                 │
│  t3                           pop() → B            ptr → C     │
│                               (B 해제됨)                        │
│                                                                 │
│  t4                           push(A')             ptr → A'    │
│                               (A의 메모리 재사용!)              │
│                               (A'의 next = C)                   │
│                                                                 │
│  t5     CAS(ptr, A, B)                                         │
│         A' == A 이므로 성공!   │                               │
│         ptr → B               │                    ptr → B     │
│                               │                    (CRASH!)    │
│                               │                                 │
│         문제: B는 이미 해제됨!│                                 │
│               또는 다른 곳에서 사용 중!                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Lock-Free Stack에서의 ABA

```c
// 취약한 Lock-Free Stack
typedef struct Node {
    void* data;
    struct Node* next;
} Node;

_Atomic(Node*) top;

void* pop() {
    Node* old_top = atomic_load(&top);

    while (old_top != NULL) {
        Node* new_top = old_top->next;  // ← 위험! old_top이 해제되었을 수 있음

        if (atomic_compare_exchange_weak(&top, &old_top, new_top)) {
            void* data = old_top->data;
            free(old_top);  // ← 다른 스레드가 아직 참조 중일 수 있음!
            return data;
        }
    }
    return NULL;
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 Stack ABA 문제 상세                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  초기 상태:                                                      │
│  top → [A|•] → [B|•] → [C|•] → NULL                            │
│         ↑                                                       │
│         T1이 읽은 위치 (old_top = A, new_top = B)               │
│                                                                 │
│  T2가 A, B를 pop:                                               │
│  top → [C|•] → NULL                                            │
│  A, B는 free됨                                                  │
│                                                                 │
│  T2가 새 노드 push (A의 메모리 재사용):                          │
│  top → [A'|•] → [C|•] → NULL                                   │
│         ↑                                                       │
│         같은 주소! (malloc이 A의 메모리 반환)                    │
│                                                                 │
│  T1의 CAS 실행:                                                  │
│  CAS(top, A, B)                                                 │
│  A' == A (같은 주소) → 성공!                                    │
│                                                                 │
│  결과:                                                           │
│  top → [B|?] → ???  (B는 이미 해제됨!)                          │
│                                                                 │
│  결과: 메모리 오염, 크래시, 데이터 손실                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 실제 발생 시나리오

### 1. 메모리 재사용 (가장 흔함)

```c
// 시나리오: Memory Allocator가 해제된 메모리를 재사용

void* vulnerable_pop() {
    Node* expected = atomic_load(&top);
    while (expected != NULL) {
        // 여기서 다른 스레드가:
        // 1. expected 노드를 pop하고 free
        // 2. 새 노드를 push (malloc이 같은 주소 반환)

        Node* next = expected->next;  // Use-After-Free!

        if (atomic_compare_exchange_weak(&top, &expected, next)) {
            return expected;
        }
    }
    return NULL;
}
```

### 2. 순환 큐/풀에서의 인덱스 재사용

```c
// 순환 버퍼에서 인덱스가 한 바퀴 돌아서 같은 값이 됨
#define QUEUE_SIZE 1024

typedef struct {
    _Atomic(uint32_t) head;
    _Atomic(uint32_t) tail;
    void* buffer[QUEUE_SIZE];
} CircularQueue;

// head가 0 → 1023 → 0 으로 순환
// 다른 스레드가 1024번 연산하면 같은 인덱스로 돌아옴
```

### 3. Lock-Free List의 노드 재삽입

```c
// 노드를 삭제했다가 다시 삽입하는 경우
void move_node(List* from, List* to, Node* node) {
    remove(from, node);  // 리스트에서 제거
    // 여기서 다른 스레드가 같은 노드를 보고 있다면?
    insert(to, node);    // 다른 리스트에 삽입
}
```

---

## 해결책

### 1. Tagged Pointer (버전 카운터)

포인터와 함께 버전 번호를 저장하여 변경을 감지합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tagged Pointer 구조                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  일반 포인터 (64-bit):                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │         64-bit pointer value                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Tagged Pointer:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 48-bit pointer │ 16-bit tag (version)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  또는 128-bit 구조체 (CMPXCHG16B):                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 64-bit pointer │ 64-bit tag                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  동작:                                                           │
│  - push/pop마다 tag 증가                                        │
│  - CAS 시 pointer + tag 모두 비교                               │
│  - 같은 포인터라도 tag가 다르면 CAS 실패                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```c
#include <stdatomic.h>
#include <stdint.h>

// 128-bit Tagged Pointer (x64 CMPXCHG16B 필요)
typedef struct {
    void* ptr;
    uint64_t tag;
} TaggedPtr;

typedef struct {
    _Atomic(TaggedPtr) top;
} TaggedStack;

void tagged_push(TaggedStack* stack, Node* node) {
    TaggedPtr expected = atomic_load(&stack->top);
    TaggedPtr desired;

    do {
        node->next = expected.ptr;
        desired.ptr = node;
        desired.tag = expected.tag + 1;  // 태그 증가
    } while (!atomic_compare_exchange_weak(&stack->top, &expected, desired));
}

Node* tagged_pop(TaggedStack* stack) {
    TaggedPtr expected = atomic_load(&stack->top);
    TaggedPtr desired;

    while (expected.ptr != NULL) {
        Node* node = expected.ptr;
        desired.ptr = node->next;
        desired.tag = expected.tag + 1;  // 태그 증가

        if (atomic_compare_exchange_weak(&stack->top, &expected, desired)) {
            return node;
        }
    }
    return NULL;
}
```

### 2. Hazard Pointers

스레드가 현재 접근 중인 포인터를 공개하여 다른 스레드가 해제하지 못하게 합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hazard Pointers 개념                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  각 스레드가 "나 이 포인터 쓰고 있어!" 선언                       │
│                                                                 │
│  Thread 1         Thread 2         Thread 3                     │
│  ┌─────────┐     ┌─────────┐     ┌─────────┐                   │
│  │ HP: A   │     │ HP: B   │     │ HP: NULL│                   │
│  └─────────┘     └─────────┘     └─────────┘                   │
│                                                                 │
│  노드 해제 시:                                                   │
│  1. 모든 HP 확인                                                │
│  2. HP에 있으면 → 나중에 해제 (retired list에 추가)              │
│  3. HP에 없으면 → 안전하게 해제                                  │
│                                                                 │
│  장점: ABA 문제 완전 해결, 메모리 누수 없음                       │
│  단점: 오버헤드 (HP 스캔), 구현 복잡                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Epoch-Based Reclamation (EBR)

스레드를 "epoch"으로 그룹화하여 안전하게 메모리를 회수합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Epoch-Based Reclamation                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  전역 Epoch: 0, 1, 2 (순환)                                     │
│                                                                 │
│  Epoch 0        Epoch 1        Epoch 2                          │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                       │
│  │ T1: 활성│   │ T2: 활성│   │         │                       │
│  │ T3: 활성│   │         │   │         │                       │
│  └─────────┘   └─────────┘   └─────────┘                       │
│                                                                 │
│  retired[0]    retired[1]    retired[2]                         │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                       │
│  │ Node A  │   │ Node B  │   │         │                       │
│  │ Node C  │   │         │   │         │                       │
│  └─────────┘   └─────────┘   └─────────┘                       │
│                                                                 │
│  규칙:                                                           │
│  - 노드 삭제 시 현재 epoch의 retired list에 추가                 │
│  - epoch e의 retired list 해제 조건:                            │
│    모든 스레드가 epoch e 이후로 넘어갔을 때                       │
│                                                                 │
│  장점: Hazard Pointers보다 오버헤드 적음                         │
│  단점: 긴 작업이 메모리 해제를 지연시킬 수 있음                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4. 메모리 풀 사용 (No-Free)

메모리를 해제하지 않고 재사용하는 전용 풀 사용.

```c
// 노드를 절대 free하지 않고 풀에서 관리
typedef struct {
    _Atomic(Node*) free_list;
} NodePool;

Node* pool_alloc(NodePool* pool) {
    Node* node = atomic_load(&pool->free_list);
    while (node != NULL) {
        Node* next = node->next;
        if (atomic_compare_exchange_weak(&pool->free_list, &node, next)) {
            return node;
        }
    }
    // free_list가 비면 새로 할당
    return malloc(sizeof(Node));
}

void pool_free(NodePool* pool, Node* node) {
    // 실제로 free하지 않고 free_list에 반환
    Node* old_head = atomic_load(&pool->free_list);
    do {
        node->next = old_head;
    } while (!atomic_compare_exchange_weak(&pool->free_list, &old_head, node));
}
```

### 5. LL/SC 하드웨어 (ARM, PowerPC)

Load-Linked/Store-Conditional은 ABA에 면역입니다.

```c
// ARM LL/SC는 "주소에 대한 접근"을 감지, 값이 아님
// A → B → A 변경 시에도 SC가 실패함

// 의사 코드
Node* pop_llsc() {
    Node* expected;
    Node* next;
    do {
        expected = LOAD_LINKED(&top);  // 모니터 설정
        if (expected == NULL) return NULL;
        next = expected->next;
    } while (!STORE_CONDITIONAL(&top, next));
    // A→B→A 변경이 있었다면 SC 실패
    return expected;
}
```

---

## 구현 예제

### Tagged Pointer Stack (완전한 구현)

```c
#include <stdatomic.h>
#include <stdint.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct Node {
    void* data;
    struct Node* next;
} Node;

// 포인터 하위 비트를 태그로 사용 (정렬된 포인터 가정)
// 또는 별도의 128-bit 구조체 사용

// 방법 1: 포인터 하위 비트 사용 (간단하지만 제한적)
#define TAG_BITS 3
#define TAG_MASK ((1ULL << TAG_BITS) - 1)
#define PTR_MASK (~TAG_MASK)

typedef uintptr_t TaggedPtr;

static inline TaggedPtr make_tagged(Node* ptr, unsigned tag) {
    return ((uintptr_t)ptr & PTR_MASK) | (tag & TAG_MASK);
}

static inline Node* get_ptr(TaggedPtr tp) {
    return (Node*)(tp & PTR_MASK);
}

static inline unsigned get_tag(TaggedPtr tp) {
    return tp & TAG_MASK;
}

// 방법 2: 128-bit 구조체 (더 많은 태그 비트)
typedef struct {
    Node* ptr;
    uint64_t tag;
} TaggedPtr128 __attribute__((aligned(16)));

typedef struct {
    _Atomic(TaggedPtr128) top;
} TaggedStack;

void tagged_stack_init(TaggedStack* stack) {
    TaggedPtr128 init = {NULL, 0};
    atomic_store(&stack->top, init);
}

void tagged_push(TaggedStack* stack, void* data) {
    Node* node = (Node*)malloc(sizeof(Node));
    node->data = data;

    TaggedPtr128 expected = atomic_load(&stack->top);
    TaggedPtr128 desired;

    do {
        node->next = expected.ptr;
        desired.ptr = node;
        desired.tag = expected.tag + 1;
    } while (!atomic_compare_exchange_weak(&stack->top, &expected, desired));
}

void* tagged_pop(TaggedStack* stack) {
    TaggedPtr128 expected = atomic_load(&stack->top);
    TaggedPtr128 desired;

    while (expected.ptr != NULL) {
        Node* node = expected.ptr;
        desired.ptr = node->next;
        desired.tag = expected.tag + 1;

        if (atomic_compare_exchange_weak(&stack->top, &expected, desired)) {
            void* data = node->data;
            free(node);  // 태그 덕분에 안전하게 해제 가능
            return data;
        }
    }
    return NULL;
}
```

### Hazard Pointer 기반 Stack (개념적 구현)

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdbool.h>

#define MAX_THREADS 64
#define HP_PER_THREAD 2

typedef struct Node {
    void* data;
    struct Node* next;
} Node;

// 전역 Hazard Pointer 배열
_Atomic(Node*) hazard_pointers[MAX_THREADS][HP_PER_THREAD];

// 스레드별 retired 리스트
typedef struct RetiredNode {
    Node* node;
    struct RetiredNode* next;
} RetiredNode;

__thread RetiredNode* retired_list = NULL;
__thread int thread_id = -1;

// HP 설정
void set_hp(int slot, Node* ptr) {
    atomic_store(&hazard_pointers[thread_id][slot], ptr);
}

// HP 해제
void clear_hp(int slot) {
    atomic_store(&hazard_pointers[thread_id][slot], NULL);
}

// 노드가 HP에 있는지 확인
bool is_protected(Node* node) {
    for (int t = 0; t < MAX_THREADS; t++) {
        for (int h = 0; h < HP_PER_THREAD; h++) {
            if (atomic_load(&hazard_pointers[t][h]) == node) {
                return true;
            }
        }
    }
    return false;
}

// retired 리스트에 추가
void retire(Node* node) {
    RetiredNode* rn = malloc(sizeof(RetiredNode));
    rn->node = node;
    rn->next = retired_list;
    retired_list = rn;

    // 주기적으로 정리 시도
    scan_and_reclaim();
}

// 안전한 노드만 해제
void scan_and_reclaim() {
    RetiredNode** curr = &retired_list;
    while (*curr != NULL) {
        if (!is_protected((*curr)->node)) {
            RetiredNode* to_free = *curr;
            *curr = (*curr)->next;
            free(to_free->node);
            free(to_free);
        } else {
            curr = &(*curr)->next;
        }
    }
}

// HP 기반 Pop
typedef struct {
    _Atomic(Node*) top;
} HPStack;

void* hp_pop(HPStack* stack) {
    Node* expected;

    while (true) {
        expected = atomic_load(&stack->top);
        if (expected == NULL) return NULL;

        // Hazard Pointer 설정
        set_hp(0, expected);

        // Double-check: top이 바뀌지 않았는지 확인
        if (atomic_load(&stack->top) != expected) {
            continue;  // 바뀌었으면 재시도
        }

        Node* next = expected->next;

        if (atomic_compare_exchange_weak(&stack->top, &expected, next)) {
            void* data = expected->data;
            clear_hp(0);
            retire(expected);  // 나중에 안전하게 해제
            return data;
        }
    }
}
```

---

## 언어별 지원

### C++ (std::atomic)

```cpp
#include <atomic>

// C++20: std::atomic<std::shared_ptr<T>> 지원
// 참조 카운팅으로 ABA 문제 해결
#include <memory>

std::atomic<std::shared_ptr<Node>> head;

void push(int value) {
    auto new_node = std::make_shared<Node>(value);
    auto old_head = head.load();
    do {
        new_node->next = old_head;
    } while (!head.compare_exchange_weak(old_head, new_node));
}

std::shared_ptr<Node> pop() {
    auto old_head = head.load();
    while (old_head && !head.compare_exchange_weak(old_head, old_head->next));
    return old_head;
}
```

### Rust (crossbeam)

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::sync::atomic::Ordering;

struct Stack<T> {
    head: Atomic<Node<T>>,
}

impl<T> Stack<T> {
    fn push(&self, data: T) {
        let guard = epoch::pin();  // Epoch 진입
        let new_node = Owned::new(Node { data, next: Atomic::null() });

        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            new_node.next.store(head, Ordering::Relaxed);

            match self.head.compare_exchange(
                head,
                new_node,
                Ordering::Release,
                Ordering::Relaxed,
                &guard,
            ) {
                Ok(_) => break,
                Err(e) => new_node = e.new,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        let guard = epoch::pin();  // Epoch 진입

        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            let head_ref = unsafe { head.as_ref()? };

            let next = head_ref.next.load(Ordering::Relaxed, &guard);

            if self.head.compare_exchange(
                head,
                next,
                Ordering::Release,
                Ordering::Relaxed,
                &guard,
            ).is_ok() {
                unsafe {
                    guard.defer_destroy(head);  // Epoch 기반 해제
                    return Some(std::ptr::read(&head_ref.data));
                }
            }
        }
    }
}
```

### Java (AtomicStampedReference)

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABASafeStack<T> {
    private static class Node<T> {
        T data;
        Node<T> next;
        Node(T data) { this.data = data; }
    }

    // 스탬프(버전)와 함께 저장
    private AtomicStampedReference<Node<T>> top =
        new AtomicStampedReference<>(null, 0);

    public void push(T data) {
        Node<T> newNode = new Node<>(data);
        int[] stampHolder = new int[1];

        while (true) {
            Node<T> oldTop = top.get(stampHolder);
            int oldStamp = stampHolder[0];
            newNode.next = oldTop;

            if (top.compareAndSet(oldTop, newNode, oldStamp, oldStamp + 1)) {
                return;
            }
        }
    }

    public T pop() {
        int[] stampHolder = new int[1];

        while (true) {
            Node<T> oldTop = top.get(stampHolder);
            if (oldTop == null) return null;

            int oldStamp = stampHolder[0];
            Node<T> newTop = oldTop.next;

            if (top.compareAndSet(oldTop, newTop, oldStamp, oldStamp + 1)) {
                return oldTop.data;
            }
        }
    }
}
```

---

## 요약

### ABA 문제 핵심

| 항목 | 설명 |
|------|------|
| **원인** | CAS가 값만 비교하고 "변경 여부"는 모름 |
| **발생 조건** | 메모리 재사용 + 동일 주소로 CAS |
| **결과** | 데이터 손상, 크래시, 메모리 오염 |

### 해결책 비교

| 해결책 | 오버헤드 | 복잡도 | 메모리 | 사용 사례 |
|--------|---------|--------|--------|----------|
| Tagged Pointer | 낮음 | 낮음 | 추가 없음 | 범용 |
| Hazard Pointers | 중간 | 높음 | HP 배열 | 정밀 제어 필요 |
| Epoch-Based | 낮음 | 중간 | retired list | Rust crossbeam |
| Memory Pool | 낮음 | 낮음 | 메모리 누수 | 제한된 환경 |
| LL/SC (HW) | 없음 | 없음 | 없음 | ARM/RISC-V |

### 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│                    ABA 해결책 선택                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  x86에서 간단히?                                                 │
│      └── Tagged Pointer (CMPXCHG16B)                           │
│                                                                 │
│  ARM/RISC-V에서?                                                 │
│      └── LL/SC 자동 처리 (대부분)                               │
│                                                                 │
│  Rust 사용?                                                      │
│      └── crossbeam (Epoch-based)                               │
│                                                                 │
│  Java 사용?                                                      │
│      └── AtomicStampedReference                                │
│                                                                 │
│  최대 성능 필요?                                                 │
│      └── Memory Pool (no-free)                                 │
│                                                                 │
│  범용 라이브러리 작성?                                           │
│      └── Hazard Pointers (가장 정밀)                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 내부 메커니즘

### CMPXCHG16B 명령어 상세

x86-64에서 128비트 CAS를 위한 명령어:

```asm
; CMPXCHG16B - 128비트 Compare-And-Swap
;
; 입력:
;   RDX:RAX = expected (128-bit)
;   RCX:RBX = desired (128-bit)
;   [memory] = 비교 대상
;
; 출력:
;   ZF=1 (성공): [memory] = RCX:RBX
;   ZF=0 (실패): RDX:RAX = [memory]

tagged_cas:
    ; TaggedPtr { ptr: RBX, tag: RCX }
    ; expected in RDX:RAX, desired in RCX:RBX

    lock cmpxchg16b [rdi]    ; 16바이트 원자적 교환
    jnz   .retry             ; ZF=0이면 실패, 재시도
    ret

.retry:
    ; RDX:RAX에 실제 값이 로드됨
    ; 재시도 로직
    jmp   tagged_cas
```

**CMPXCHG16B 요구사항**:
```
┌─────────────────────────────────────────────────────────────────────┐
│  CPU Feature:  CMPXCHG16B (CPUID.01H:ECX[13])                       │
│  Alignment:    16바이트 정렬 필수 (misaligned → #GP 예외)            │
│  Performance:  CMPXCHG보다 ~2배 느림 (더 넓은 버스 락)               │
│                                                                     │
│  컴파일러 지원:                                                      │
│  - GCC/Clang: -mcx16 플래그 필요                                     │
│  - MSVC: _InterlockedCompareExchange128                             │
│  - C++: std::atomic<T> 크기 16바이트면 자동 사용                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 메모리 할당자와 ABA 발생 원리

```
┌─────────────────────────────────────────────────────────────────────┐
│                    malloc/free와 ABA 발생                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  대부분의 메모리 할당자는 "free list" 사용                            │
│                                                                     │
│  Step 1: malloc(sizeof(Node)) → 주소 0x1000 반환                    │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Free List: [0x2000] → [0x3000] → NULL                      │     │
│  │ 사용 중:   [0x1000] (Node A)                                │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Step 2: free(0x1000) → 0x1000이 free list 앞에 추가               │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Free List: [0x1000] → [0x2000] → [0x3000] → NULL           │     │
│  │ 사용 중:   (없음)                                           │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Step 3: malloc(sizeof(Node)) → 0x1000 다시 반환!                   │
│  ┌────────────────────────────────────────────────────────────┐     │
│  │ Free List: [0x2000] → [0x3000] → NULL                      │     │
│  │ 사용 중:   [0x1000] (Node B, 같은 주소!)                    │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  → CAS는 주소만 비교: 0x1000 == 0x1000 (성공!)                      │
│  → 하지만 내용(Node A vs Node B)은 완전히 다름                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**TCMalloc/jemalloc의 Thread-Local Cache**:
```
더 빠른 ABA 발생 가능:
- Thread-Local Free List는 더 빠르게 순환
- 같은 스레드의 free → malloc이 바로 같은 주소 반환
- ABA 발생 확률 증가
```

### LL/SC (Load-Linked/Store-Conditional) 동작

ARM/RISC-V에서 ABA에 면역인 이유:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LL/SC vs CAS 비교                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CAS (x86 CMPXCHG):                                                 │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  if (*addr == expected) {                                   │    │
│  │      *addr = desired;                                       │    │
│  │      return SUCCESS;                                        │    │
│  │  } else {                                                   │    │
│  │      return FAILURE;                                        │    │
│  │  }                                                          │    │
│  │                                                             │    │
│  │  → "값"만 비교                                               │    │
│  │  → A→B→A 변경 감지 불가                                      │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  LL/SC (ARM LDXR/STXR, RISC-V LR/SC):                               │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  LL: value = *addr; SET_MONITOR(addr);                      │    │
│  │                                                             │    │
│  │  SC: if (MONITOR_VALID(addr)) {  // 주소에 대한 접근 감지    │    │
│  │          *addr = desired;                                   │    │
│  │          CLEAR_MONITOR();                                   │    │
│  │          return SUCCESS;                                    │    │
│  │      } else {                                               │    │
│  │          return FAILURE;                                    │    │
│  │      }                                                      │    │
│  │                                                             │    │
│  │  → "주소에 대한 접근"을 감지                                  │    │
│  │  → A→B→A 변경 시 모니터가 클리어되어 SC 실패                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**ARM의 Exclusive Monitor 구현**:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    ARM Exclusive Monitor                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Local Monitor (각 CPU에 존재):                                      │
│  ┌──────────────────────────────────────────┐                       │
│  │  state: OPEN / EXCLUSIVE                 │                       │
│  │  address: 모니터링 중인 주소             │                       │
│  │  granule: 보통 캐시 라인 크기            │                       │
│  └──────────────────────────────────────────┘                       │
│                                                                     │
│  Global Monitor (시스템 버스에 존재):                                │
│  ┌──────────────────────────────────────────┐                       │
│  │  각 CPU의 exclusive 요청 추적            │                       │
│  │  다른 CPU의 쓰기 시 해당 CPU 모니터 클리어│                       │
│  └──────────────────────────────────────────┘                       │
│                                                                     │
│  동작 예:                                                            │
│  CPU0: LDXR x0, [addr]  → Local Monitor = EXCLUSIVE                 │
│  CPU1: STR  x1, [addr]  → CPU0의 Monitor = OPEN (스눕)              │
│  CPU0: STXR w2, x3, [addr] → FAIL (w2 = 1)                          │
│                                                                     │
│  → 중간에 "어떤 쓰기라도" 있으면 SC 실패                              │
│  → ABA 문제 자연스럽게 해결                                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tagged Pointer 오버플로우 고려

```cpp
// 16비트 태그: 65536번 연산 후 오버플로우
struct TaggedPtr {
    void* ptr;        // 48비트 (실제 사용)
    uint16_t tag;     // 16비트
};

// 오버플로우 시나리오:
// t=0: {ptr=A, tag=0}
// ... 65535번 연산 ...
// t=65535: {ptr=?, tag=65535}
// t=65536: {ptr=A, tag=0}  ← 처음과 동일!

// 해결책:
// 1. 64비트 태그 사용 (128비트 CAS)
// 2. 태그 오버플로우 확률 계산:
//    - 초당 1백만 연산 기준
//    - 16비트: ~0.065초에 순환
//    - 32비트: ~71분에 순환
//    - 64비트: ~584,942년에 순환 (사실상 안전)
```

### ABA 탐지 도구

```bash
# ThreadSanitizer는 ABA를 직접 탐지하지 못함
# (값이 같으면 race로 보지 않음)

# 대신 Use-After-Free를 탐지:
$ clang++ -fsanitize=address -g aba_test.cpp -o aba_test
$ ./aba_test

# AddressSanitizer 출력:
# ==12345==ERROR: AddressSanitizer: heap-use-after-free
# READ of size 8 at 0x... thread T1
#     #0 0x... in pop() aba_test.cpp:42
#     #1 ...
#
# 0x... is located 0 bytes inside of 64-byte region
# freed by thread T2 here:
#     #0 0x... in free
#     #1 0x... in pop() aba_test.cpp:50

# Valgrind도 Use-After-Free 탐지:
$ valgrind --tool=memcheck ./aba_test
```

---

## 관련 문서

- [CAS 연산](./01-cas-operation.md) - CAS의 기본 동작
- [Hazard Pointers](./08-hazard-pointers.md) - HP 상세 구현
- [Lock-Free Stack](./03-lock-free-stack.md) - ABA 문제가 발생하는 예
- [Lock-Free Queue](./04-lock-free-queue.md) - Michael-Scott Queue의 ABA 처리

---

## 참고 자료

- [ABA Problem - Wikipedia](https://en.wikipedia.org/wiki/ABA_problem)
- [Lock-Free Data Structures with Hazard Pointers - Dr. Dobb's](https://www.drdobbs.com/lock-free-data-structures-with-hazard-lk/184401890)
- [Crossbeam Epoch - Rust Documentation](https://docs.rs/crossbeam-epoch/latest/crossbeam_epoch/)
- [C++ Concurrency in Action, Chapter 7](https://www.manning.com/books/c-plus-plus-concurrency-in-action)
