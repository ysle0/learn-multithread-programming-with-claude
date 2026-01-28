# Lock-Free Queue (Michael-Scott Queue)

## 📌 개요

**Michael-Scott Queue**는 1996년 Maged M. Michael과 Michael L. Scott이 발표한 Lock-Free FIFO 큐입니다. 현재까지도 가장 널리 사용되는 Lock-Free 큐 알고리즘이며, Java의 `ConcurrentLinkedQueue`, C++의 `boost::lockfree::queue` 등의 기반이 됩니다.

---

## 🎯 핵심 아이디어

스택과 달리 큐는 **두 개의 atomic 포인터**가 필요합니다:

1. **Head**: 큐의 앞 (dequeue 위치)
2. **Tail**: 큐의 뒤 (enqueue 위치)

```
Initial State (더미 노드 사용):
head → [dummy] ← tail
        ↓
       NULL

After enqueue(1):
head → [dummy] → [1] ← tail
                  ↓
                 NULL

After enqueue(2):
head → [dummy] → [1] → [2] ← tail
                        ↓
                       NULL

After dequeue():
       head → [1] → [2] ← tail
               ↓
              NULL
```

**핵심**: Dummy 노드를 사용하여 빈 큐와 1개 요소 큐를 통일되게 처리

---

## 📝 의사코드 (Pseudocode)

### 자료구조 정의

```
struct Node {
    T data;
    atomic<Node*> next;

    Node(T value) {
        data = value
        next = NULL
    }
}

class LockFreeQueue {
private:
    atomic<Node*> head;    // 큐의 앞
    atomic<Node*> tail;    // 큐의 뒤

public:
    LockFreeQueue() {
        // 더미 노드 생성
        dummy = allocate new Node
        head = dummy
        tail = dummy
    }
}
```

### Enqueue 연산

```
procedure Enqueue(value: T)
    new_node = allocate new Node(value)

    loop
        last = tail.load(memory_order_acquire)
        next = last.next.load(memory_order_acquire)

        // tail이 변경되지 않았는지 확인
        if last == tail.load(memory_order_acquire) then

            if next == NULL then
                // Case 1: tail이 실제 마지막 노드
                // last.next를 new_node로 설정 시도
                if compare_exchange_weak(last.next, NULL, new_node,
                                         memory_order_release,
                                         memory_order_acquire) then
                    // 성공: tail을 새 노드로 이동 시도 (실패해도 괜찮음)
                    compare_exchange_weak(tail, last, new_node,
                                          memory_order_release,
                                          memory_order_acquire)
                    return
                end if
            else
                // Case 2: tail이 뒤처짐
                // 다른 스레드가 enqueue 중인 상태
                // tail을 도와서 앞으로 이동
                compare_exchange_weak(tail, last, next,
                                      memory_order_release,
                                      memory_order_acquire)
            end if
        end if
    end loop
end procedure
```

**동작 과정**:
1. 새 노드 할당
2. 현재 tail과 tail.next 읽기
3. **Case 1**: tail.next가 NULL → tail이 실제 마지막
   - tail.next를 new_node로 CAS
   - 성공하면 tail을 new_node로 이동 (실패해도 OK)
4. **Case 2**: tail.next가 NULL이 아님 → tail이 뒤처짐
   - tail을 next로 이동 (다른 스레드 도움)
   - 재시도

### Dequeue 연산

```
procedure Dequeue() -> T
    loop
        first = head.load(memory_order_acquire)
        last = tail.load(memory_order_acquire)
        next = first.next.load(memory_order_acquire)

        // head가 변경되지 않았는지 확인
        if first == head.load(memory_order_acquire) then

            if first == last then
                // 큐가 비어있거나 tail이 뒤처짐
                if next == NULL then
                    throw QueueEmptyException  // 큐가 비어있음
                end if

                // tail이 뒤처짐 → tail을 앞으로 이동
                compare_exchange_weak(tail, last, next,
                                      memory_order_release,
                                      memory_order_acquire)
            else
                // 큐에 요소가 있음
                result = next.data  // 더미 노드 다음 값

                // head를 next로 이동 시도
                if compare_exchange_weak(head, first, next,
                                         memory_order_release,
                                         memory_order_acquire) then
                    delete first  // ⚠️ 안전한 메모리 회수 필요
                    return result
                end if
            end if
        end if
    end loop
end procedure
```

**동작 과정**:
1. head, tail, head.next 읽기
2. **빈 큐 체크**: head == tail && next == NULL
3. **tail 뒤처짐 체크**: head == tail && next != NULL → tail 이동
4. **정상 dequeue**: head를 next로 CAS
   - 성공하면 이전 head(더미) 제거, next.data 반환

---

## 💻 C++ 구현 예제

```cpp
#include <atomic>
#include <optional>

template<typename T>
class MichaelScottQueue {
private:
    struct Node {
        T data;
        std::atomic<Node*> next;

        Node() : next(nullptr) {}  // 더미 노드용
        Node(const T& value) : data(value), next(nullptr) {}
    };

    std::atomic<Node*> head;
    std::atomic<Node*> tail;

public:
    MichaelScottQueue() {
        Node* dummy = new Node();
        head.store(dummy, std::memory_order_relaxed);
        tail.store(dummy, std::memory_order_relaxed);
    }

    ~MichaelScottQueue() {
        while (dequeue()) {}  // 모든 요소 제거
        delete head.load();    // 더미 노드 제거
    }

    void enqueue(const T& value) {
        Node* new_node = new Node(value);

        while (true) {
            Node* last = tail.load(std::memory_order_acquire);
            Node* next = last->next.load(std::memory_order_acquire);

            // tail이 변경되지 않았는지 재확인
            if (last == tail.load(std::memory_order_acquire)) {

                if (next == nullptr) {
                    // Case 1: tail이 실제 마지막
                    if (last->next.compare_exchange_weak(
                        next, new_node,
                        std::memory_order_release,
                        std::memory_order_acquire)) {

                        // tail 이동 시도 (실패해도 괜찮음)
                        tail.compare_exchange_weak(
                            last, new_node,
                            std::memory_order_release,
                            std::memory_order_acquire);
                        return;
                    }
                } else {
                    // Case 2: tail이 뒤처짐 → 도움
                    tail.compare_exchange_weak(
                        last, next,
                        std::memory_order_release,
                        std::memory_order_acquire);
                }
            }
        }
    }

    std::optional<T> dequeue() {
        while (true) {
            Node* first = head.load(std::memory_order_acquire);
            Node* last = tail.load(std::memory_order_acquire);
            Node* next = first->next.load(std::memory_order_acquire);

            // head가 변경되지 않았는지 재확인
            if (first == head.load(std::memory_order_acquire)) {

                if (first == last) {
                    if (next == nullptr) {
                        return std::nullopt;  // 큐가 비어있음
                    }

                    // tail이 뒤처짐 → 도움
                    tail.compare_exchange_weak(
                        last, next,
                        std::memory_order_release,
                        std::memory_order_acquire);
                } else {
                    T result = next->data;

                    if (head.compare_exchange_weak(
                        first, next,
                        std::memory_order_release,
                        std::memory_order_acquire)) {

                        delete first;  // ⚠️ 안전한 메모리 회수 필요
                        return result;
                    }
                }
            }
        }
    }

    bool empty() const {
        Node* first = head.load(std::memory_order_acquire);
        Node* next = first->next.load(std::memory_order_acquire);
        return next == nullptr;
    }
};
```

---

## 🔍 상세 분석

### 왜 Dummy 노드가 필요한가?

Dummy 노드 없이 구현하면:
- **빈 큐**: head == NULL, tail == NULL
- **1개 요소**: head == tail == [node]

이 경우 enqueue와 dequeue가 동시에 head와 tail을 수정하려고 하면 복잡해집니다.

Dummy 노드 사용 시:
- **빈 큐**: head == tail == [dummy], dummy.next == NULL
- **1개 요소**: head == [dummy] → [node], tail == [node]

→ 항상 head와 tail이 분리되어 더 단순함

### Tail 뒤처짐 (Lagging Tail)

```
상황: Thread 1이 enqueue 중

1. Thread 1이 tail.next를 new_node로 설정 (CAS 성공)
2. (Thread 1이 일시 중지)
3. 현재 상태:
   tail → [old] → [new_node]
                   ↑
                 여기가 실제 마지막이지만 tail이 아직 업데이트 안됨

4. Thread 2가 enqueue 시도:
   - tail.next != NULL 발견
   - tail을 next로 이동 (Thread 1 도움)
   - 그 다음 자신의 enqueue 진행
```

이 "도움" 메커니즘이 Lock-Free 보장의 핵심입니다!

### Memory Ordering

| 연산 | Memory Order | 이유 |
|------|--------------|------|
| enqueue load | `acquire` | next 포인터 읽기 전 동기화 |
| enqueue CAS (성공) | `release` | 새 노드를 다른 스레드에 보이게 |
| dequeue load | `acquire` | 데이터 읽기 전 동기화 |
| dequeue CAS (성공) | `release` | head 변경을 다른 스레드에 보이게 |

---

## 📊 성능 특성

| 특성 | 값 |
|------|-----|
| **Enqueue 시간 복잡도** | O(1) 평균 (재시도 제외) |
| **Dequeue 시간 복잡도** | O(1) 평균 |
| **공간 복잡도** | O(n) + O(1) (더미 노드) |
| **Progress Guarantee** | Lock-Free |
| **확장성** | 매우 우수 (head/tail 분리) |

### 장점
- ✅ **Head와 Tail 분리**: Enqueue와 Dequeue가 서로 다른 위치 접근 → 경합 감소
- ✅ **Lock-Free**: 데드락 없음
- ✅ **확장성**: 스레드 수 증가에도 성능 유지

### 단점
- ⚠️ **ABA 문제**: 포인터 재사용 위험
- ⚠️ **메모리 회수 복잡**: Hazard Pointers 등 필요
- ⚠️ **False Sharing**: head와 tail이 같은 캐시 라인에 있으면 성능 저하

**False Sharing 해결**:
```cpp
struct alignas(64) AlignedPointer {
    std::atomic<Node*> ptr;
    char padding[64 - sizeof(std::atomic<Node*>)];
};

AlignedPointer head;  // 별도 캐시 라인
AlignedPointer tail;  // 별도 캐시 라인
```

---

## 🔄 변형: MPMC Queue

Michael-Scott Queue는 **다중 생산자-다중 소비자(MPMC)** 환경에서 사용됩니다.

**단일 생산자-단일 소비자(SPSC)**라면 더 간단한 알고리즘 사용 가능:
- Ring Buffer 기반 큐
- 더 빠르고 간단함

---

## ⚠️ 주의사항

### 1. ABA 문제
```cpp
// Thread 1: dequeue 시작, first 읽음
Node* first = head.load();  // [A]

// Thread 2: dequeue [A], dequeue [B], enqueue [A] (재사용)
// 이제 head는 다시 [A]지만 내용이 다름!

// Thread 1: CAS 성공 (잘못된 성공!)
head.compare_exchange_weak(first, next);  // 위험!
```

**해결**: Tagged Pointers, Hazard Pointers

### 2. 메모리 회수
```cpp
delete first;  // ❌ 다른 스레드가 아직 접근 중일 수 있음
```

**해결**: Hazard Pointers, Epoch-based Reclamation

### 3. Spurious Failures
`compare_exchange_weak`는 가짜 실패(spurious failure) 가능 → 루프로 재시도 필요

---

## 🎯 사용 사례

### 적합한 경우
- **작업 큐**: 스레드 풀의 작업 큐
- **메시지 큐**: 스레드 간 통신
- **이벤트 큐**: 비동기 이벤트 처리
- **MPMC 시나리오**: 다중 생산자/소비자

### 부적합한 경우
- **SPSC**: 단일 생산자/소비자는 Ring Buffer가 더 빠름
- **복잡한 우선순위**: Priority Queue는 Lock-Free 구현이 매우 어려움

---

## 🔧 내부 메커니즘

### Head/Tail 분리와 캐시 라인 최적화

Michael-Scott Queue의 핵심 설계는 **Head와 Tail의 물리적 분리**입니다:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     메모리 레이아웃 (비최적화)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cache Line 0 (64 bytes)                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  head (8B)  │  tail (8B)  │  padding...                     │    │
│  └─────────────────────────────────────────────────────────────┘    │
│        ↑              ↑                                             │
│     Dequeue       Enqueue                                           │
│     Thread        Thread                                            │
│                                                                     │
│  문제: head와 tail이 같은 캐시 라인 → False Sharing!                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     메모리 레이아웃 (최적화)                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Cache Line 0 (64 bytes)           Cache Line 1 (64 bytes)          │
│  ┌─────────────────────────┐       ┌─────────────────────────┐      │
│  │  head (8B) + pad (56B)  │       │  tail (8B) + pad (56B)  │      │
│  └─────────────────────────┘       └─────────────────────────┘      │
│        ↑                                  ↑                         │
│     Dequeue                           Enqueue                       │
│     Threads                           Threads                       │
│                                                                     │
│  결과: head와 tail이 독립적으로 동작 → 경합 최소화                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**실측 성능 차이**:
```
False Sharing 있음:     ~15M ops/sec (4 threads)
False Sharing 제거:     ~45M ops/sec (4 threads)
→ 약 3배 성능 향상
```

### Helping 메커니즘의 정확한 동작

Tail 업데이트가 2단계로 분리되어 있어 "중간 상태"가 존재합니다:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Enqueue 2단계 동작                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  단계 1: tail->next CAS                                             │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                             │    │
│  │   tail ─────────────────────┐                               │    │
│  │                             ▼                               │    │
│  │   [dummy] ──→ [A] ──→ [B] ──→ [NEW]                         │    │
│  │     ↑                         ↑                             │    │
│  │   head                   tail->next = NEW (CAS 성공)         │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ⚠️ 중간 상태: tail이 실제 끝(NEW)을 가리키지 않음                    │
│                                                                     │
│  단계 2: tail CAS (또는 다른 스레드의 helping)                        │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                                                             │    │
│  │   [dummy] ──→ [A] ──→ [B] ──→ [NEW] ◀── tail                │    │
│  │     ↑                                                       │    │
│  │   head                                                      │    │
│  │                                                             │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Helping 코드의 원자성 보장**:
```cpp
// Thread A: enqueue 중단 상태
// tail -> [B], tail->next -> [NEW]

// Thread B: 새로운 enqueue 시도
Node* last = tail.load();           // last = [B]
Node* next = last->next.load();     // next = [NEW] (not NULL!)

if (next != nullptr) {
    // "tail이 뒤처졌다" 감지
    // Thread A를 대신해서 tail 이동
    tail.compare_exchange_weak(last, next);
    // 성공 여부와 관계없이 재시도
    continue;
}
```

**왜 Helping이 안전한가?**
1. tail->next CAS는 **단 한 스레드만** 성공
2. tail CAS는 **여러 스레드가** 시도할 수 있지만, 결과는 동일
3. 잘못된 위치로 tail이 이동하는 것은 불가능 (항상 next 방향)

### x86 어셈블리 분석: Enqueue CAS

```asm
; last->next.compare_exchange_weak(expected, new_node)
; expected = NULL (0), new_node = %r12

enqueue_cas:
    mov    rax, 0                    ; expected = NULL
    lock cmpxchg [rbx+8], r12        ; rbx = last, offset 8 = next 필드
    jnz    enqueue_retry             ; ZF=0이면 실패, 재시도

    ; CAS 성공 - tail 업데이트 시도
    mov    rax, rbx                  ; expected = last
    lock cmpxchg [rip+tail], r12     ; tail = new_node 시도
    ; 실패해도 OK - 다른 스레드가 도움
    ret

enqueue_retry:
    ; rax에 실제 값이 들어있음 (expected가 업데이트됨)
    ; 이 값이 NULL이 아니면 helping 필요
    test   rax, rax
    jnz    help_tail_advance
    jmp    enqueue_cas
```

### Memory Ordering 상세 분석

```cpp
// Enqueue에서의 동기화 패턴
void enqueue(T value) {
    Node* new_node = new Node(value);

    while (true) {
        // [1] acquire: 이후 읽기가 재배치되지 않음
        Node* last = tail.load(memory_order_acquire);

        // [2] acquire: last->next 읽기 동기화
        Node* next = last->next.load(memory_order_acquire);

        if (next == nullptr) {
            // [3] release: new_node의 data가 먼저 보이도록
            if (last->next.compare_exchange_weak(
                next, new_node,
                memory_order_release,    // 성공 시
                memory_order_acquire)) { // 실패 시

                // [4] release: tail 업데이트
                tail.compare_exchange_weak(
                    last, new_node,
                    memory_order_release,
                    memory_order_relaxed);  // 실패해도 무관
                return;
            }
        }
    }
}
```

**동기화 그래프**:
```
Thread A (Enqueue)              Thread B (Dequeue)
─────────────────               ─────────────────
new_node->data = X
        │
        ▼ release
last->next = new_node ─────────────→ next = first->next
                           acquire         │
                                          ▼
                                    result = next->data
                                    (X가 보임 - 보장됨!)
```

### Dequeue의 데이터 무결성

더미 노드 설계가 **dequeue 안전성**을 보장합니다:

```
상태: [dummy] -> [A] -> [B]
      head       real first

Dequeue 순서:
1. first = head (dummy)
2. next = first->next (A - 실제 첫 데이터)
3. result = next->data (A의 데이터 읽기)
4. head CAS: dummy -> A

핵심: 데이터를 읽는 시점에 해당 노드(A)는 아직 head가 아님
      → 다른 dequeue가 A를 건드리지 않음
```

**일반적인 실수 (더미 노드 없이)**:
```cpp
// ❌ 잘못된 구현 (더미 노드 없음)
T dequeue() {
    Node* first = head.load();
    T result = first->data;      // 데이터 읽기
    head.compare_exchange(first, first->next);  // head 이동
    // 문제: 데이터 읽는 동안 다른 스레드가 first를 dequeue할 수 있음!
}
```

### MESI 프로토콜과 Two-Pointer Queue

```
┌─────────────────────────────────────────────────────────────────────┐
│            Head/Tail 분리의 MESI 효과                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  CPU 0 (Dequeue)                  CPU 1 (Enqueue)                   │
│  ┌─────────────────┐              ┌─────────────────┐               │
│  │ L1 Cache        │              │ L1 Cache        │               │
│  │                 │              │                 │               │
│  │ [head: E/M]     │              │ [tail: E/M]     │               │
│  │                 │              │                 │               │
│  └────────┬────────┘              └────────┬────────┘               │
│           │                                │                        │
│           └───────────────┬────────────────┘                        │
│                           │                                         │
│                    L3 Cache / Memory                                │
│                                                                     │
│  결과:                                                               │
│  - head는 CPU 0에서 Exclusive/Modified                               │
│  - tail은 CPU 1에서 Exclusive/Modified                               │
│  - 서로 무효화하지 않음 (다른 캐시 라인)                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

대조: 단일 포인터 (Stack)
┌─────────────────────────────────────────────────────────────────────┐
│  CPU 0 (Push)                     CPU 1 (Pop)                       │
│  ┌─────────────────┐              ┌─────────────────┐               │
│  │ [top: M→I→M→I]  │   ping-     │ [top: I→M→I→M]  │               │
│  │                 │ ◀──pong──▶  │                 │               │
│  └─────────────────┘              └─────────────────┘               │
│                                                                     │
│  → 매 연산마다 캐시 라인 전송 필요                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Linked List 순회의 안전성

```cpp
// 안전한 순회 패턴
Node* current = head.load(memory_order_acquire);
while (current != nullptr) {
    // current가 유효한지 어떻게 보장하나?

    // 문제: dequeue가 current를 delete할 수 있음
    Node* next = current->next.load(memory_order_acquire);

    // 해결: Hazard Pointer 또는 EBR
    // HP: current를 HP에 등록 → delete 방지
    // EBR: Grace Period 동안 delete 지연

    current = next;
}
```

---

## 🔬 실전 최적화

### 1. Backoff 전략
```cpp
void enqueue_with_backoff(const T& value) {
    // ... CAS 실패 시
    int backoff = 1;
    for (int i = 0; i < backoff; i++) {
        std::this_thread::yield();
    }
    backoff = std::min(backoff * 2, 64);
}
```

### 2. Padding으로 False Sharing 방지
```cpp
struct alignas(std::hardware_destructive_interference_size) PaddedAtomic {
    std::atomic<Node*> ptr;
};
```

### 3. Bulk Operations
여러 요소를 한 번에 enqueue/dequeue하여 CAS 횟수 감소

---

## 🔗 관련 문서

- [Lock-Free Stack](./03-lock-free-stack.md) - 더 간단한 구조
- [ABA Problem](./07-aba-problem.md) - 큐에서의 ABA 해결
- [Hazard Pointers](./08-hazard-pointers.md) - 안전한 메모리 회수

---

## 📚 참고 자료

- Michael & Scott (1996). "Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms"
- Herlihy & Shavit. "The Art of Multiprocessor Programming" - Chapter 10

---

*Michael-Scott Queue는 Lock-Free 큐의 사실상 표준입니다. 복잡하지만 실용성이 검증되었습니다.*
