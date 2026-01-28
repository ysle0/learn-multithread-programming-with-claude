# Lock-Free Stack (Treiber Stack)

## 📌 개요

**Treiber Stack**은 1986년 R. K. Treiber가 제안한 가장 단순하고 널리 사용되는 Lock-Free 스택 구현입니다. CAS(Compare-And-Swap) 연산만으로 동시성을 제어하여 락 없이 안전한 스택 연산을 제공합니다.

---

## 🎯 핵심 아이디어

전통적인 스택은 락으로 보호되지만, Treiber Stack은 다음과 같이 작동합니다:

1. **단일 원자적 포인터**: 스택의 top을 가리키는 atomic 포인터 하나만 사용
2. **CAS 기반 업데이트**: top 포인터를 CAS로 원자적으로 교체
3. **재시도 루프**: CAS 실패 시 다시 시도

```
Initial State:
top → [3] → [2] → [1] → NULL

Push(4):
1. new_node → [4]
2. new_node.next = top (현재 [3])
3. CAS(top, [3], [4])  // top을 [3]에서 [4]로 교체

Result:
top → [4] → [3] → [2] → [1] → NULL
```

---

## 📝 의사코드 (Pseudocode)

### 자료구조 정의

```
struct Node {
    T data;              // 저장할 데이터
    Node* next;          // 다음 노드를 가리키는 포인터
}

class LockFreeStack {
private:
    atomic<Node*> top;   // 스택의 맨 위를 가리키는 atomic 포인터

public:
    LockFreeStack() {
        top = NULL;
    }
}
```

### Push 연산

```
procedure Push(value: T)
    new_node = allocate new Node
    new_node.data = value

    loop
        old_top = top.load(memory_order_relaxed)
        new_node.next = old_top

        // CAS: top이 여전히 old_top이면 new_node로 교체
        if compare_exchange_weak(top, old_top, new_node,
                                 memory_order_release,
                                 memory_order_relaxed) then
            return  // 성공
        // 실패하면 루프 반복 (다른 스레드가 먼저 수정함)
    end loop
end procedure
```

**동작 과정**:
1. 새 노드 할당 및 데이터 설정
2. 현재 top 읽기
3. 새 노드의 next를 현재 top으로 설정
4. CAS로 top을 new_node로 교체 시도
   - 성공: 반환
   - 실패: 다른 스레드가 수정함 → 재시도

### Pop 연산

```
procedure Pop() -> T
    loop
        old_top = top.load(memory_order_acquire)

        if old_top == NULL then
            throw StackEmptyException

        next_node = old_top.next

        // CAS: top이 여전히 old_top이면 next_node로 교체
        if compare_exchange_weak(top, old_top, next_node,
                                 memory_order_release,
                                 memory_order_acquire) then
            result = old_top.data
            delete old_top  // ⚠️ 메모리 회수 문제 발생 가능 (나중에 다룸)
            return result
        // 실패하면 루프 반복
    end loop
end procedure
```

**동작 과정**:
1. 현재 top 읽기
2. 스택이 비어있으면 예외
3. top의 다음 노드 저장
4. CAS로 top을 다음 노드로 교체 시도
   - 성공: 이전 top의 데이터 반환
   - 실패: 재시도

---

## 💻 C++ 구현 예제

```cpp
#include <atomic>
#include <memory>
#include <optional>

template<typename T>
class TreiberStack {
private:
    struct Node {
        T data;
        Node* next;

        Node(const T& value) : data(value), next(nullptr) {}
    };

    std::atomic<Node*> top;

public:
    TreiberStack() : top(nullptr) {}

    ~TreiberStack() {
        while (pop()) {}  // 모든 노드 제거
    }

    void push(const T& value) {
        Node* new_node = new Node(value);

        // CAS 루프
        Node* old_top = top.load(std::memory_order_relaxed);
        do {
            new_node->next = old_top;
        } while (!top.compare_exchange_weak(
            old_top, new_node,
            std::memory_order_release,
            std::memory_order_relaxed
        ));
    }

    std::optional<T> pop() {
        Node* old_top = top.load(std::memory_order_acquire);

        // CAS 루프
        while (old_top != nullptr) {
            Node* next_node = old_top->next;

            if (top.compare_exchange_weak(
                old_top, next_node,
                std::memory_order_release,
                std::memory_order_acquire
            )) {
                T result = old_top->data;
                delete old_top;  // ⚠️ 실제로는 안전한 메모리 회수 필요
                return result;
            }
        }

        return std::nullopt;  // 스택이 비어있음
    }

    bool empty() const {
        return top.load(std::memory_order_relaxed) == nullptr;
    }
};
```

---

## Internal Mechanisms

### CPU 캐시와 CAS 연산

Treiber Stack의 성능은 CPU 캐시 일관성 프로토콜에 크게 영향 받습니다.

#### MESI 상태 전이

```
Push 연산 시 top 포인터의 캐시 상태:

CPU 0 (Push)                    CPU 1 (다른 Push)
─────────────                   ─────────────────
1. top 읽기                     1. top 읽기
   캐시: Shared                    캐시: Shared
        ▼                              ▼
2. CAS 시도 (LOCK CMPXCHG)
   캐시: Modified (배타적 획득)
   → CPU 1의 캐시 라인 Invalidate
        ▼                              ▼
3. CAS 성공                     2. CAS 시도
   top 업데이트됨                  → 캐시 미스! (Invalid 상태)
                                  → CPU 0에서 캐시 라인 가져옴
                                  → CAS 실패 (값이 변경됨)

캐시 라인 ping-pong:
  고경합 시 top 포인터의 캐시 라인이
  CPU 간에 계속 이동 → 성능 저하
```

#### x86 LOCK CMPXCHG 동작

```nasm
; Treiber Stack push의 CAS 부분
; old_top이 EAX에, new_node가 EBX에 있다고 가정

retry:
    mov eax, [top]           ; top 읽기 (캐시에서)
    mov [new_node+next], eax ; new_node->next = old_top

    ; LOCK prefix로 버스 락 또는 캐시 락 획득
    lock cmpxchg [top], ebx  ; if (top == EAX) top = EBX

    jne retry                ; ZF=0이면 실패, 재시도

; LOCK 명령의 효과:
; 1. 해당 캐시 라인을 Modified 상태로 전환
; 2. 다른 CPU의 동일 캐시 라인 Invalidate
; 3. Store Buffer flush (seq_cst보다 약함)
```

#### ARM LL/SC 동작

```nasm
; ARM64 Treiber Stack push
; x0 = &top, x1 = new_node

push_retry:
    ldaxr   x2, [x0]         ; Load-Exclusive (top 읽기 + 예약)
    str     x2, [x1, #next]  ; new_node->next = old_top

    stlxr   w3, x1, [x0]     ; Store-Exclusive (조건부 저장)
                             ; w3 = 0이면 성공, 1이면 실패

    cbnz    w3, push_retry   ; 실패시 재시도

; LL/SC의 장점:
; 1. ABA 문제에 면역 (값이 아닌 접근을 감지)
; 2. 하지만 spurious failure 가능 (같은 캐시 라인의 다른 위치 쓰기로도 실패)
```

### 메모리 회수의 내부 동작

Pop에서 `delete old_top`이 위험한 이유를 하드웨어 관점에서 분석합니다.

```
Thread 1 (Pop)              Thread 2 (Pop 시도)
──────────────              ─────────────────────
1. old_top = top (= A)
   next = old_top->next     1. old_top = top (= A)
                               (같은 A를 읽음)
        ▼
2. CAS 성공, top = next
        ▼
3. delete old_top (A)
   → A의 메모리가 해제됨
                               ▼
                            2. next = old_top->next
                               → Use-After-Free!
                               → A의 메모리가 이미 해제됨
                               → old_top->next가 가비지 값

하드웨어 수준에서:
  - Thread 2의 old_top->next 읽기가 해제된 메모리 접근
  - 해당 메모리가 재할당되었다면 다른 데이터 읽음
  - 세그폴트 또는 데이터 손상 발생 가능
```

### Tagged Pointer 구현

ABA 문제 해결을 위한 Tagged Pointer의 하드웨어 지원:

```cpp
// x86-64: CMPXCHG16B를 이용한 128비트 CAS
// 포인터(64비트) + 태그(64비트)

struct alignas(16) TaggedPtr {
    Node* ptr;
    uint64_t tag;
};

std::atomic<TaggedPtr> top;

void push_tagged(Node* new_node) {
    TaggedPtr old_top = top.load(std::memory_order_relaxed);
    TaggedPtr new_top;

    do {
        new_node->next = old_top.ptr;
        new_top.ptr = new_node;
        new_top.tag = old_top.tag + 1;  // 태그 증가
    } while (!top.compare_exchange_weak(old_top, new_top,
                                        std::memory_order_release,
                                        std::memory_order_relaxed));
}

Node* pop_tagged() {
    TaggedPtr old_top = top.load(std::memory_order_acquire);
    TaggedPtr new_top;

    while (old_top.ptr != nullptr) {
        new_top.ptr = old_top.ptr->next;
        new_top.tag = old_top.tag + 1;

        if (top.compare_exchange_weak(old_top, new_top,
                                      std::memory_order_release,
                                      std::memory_order_acquire)) {
            return old_top.ptr;
        }
    }
    return nullptr;
}

/*
 * ABA 방지 원리:
 *
 * Thread 1:  old = {A, tag=5}
 * Thread 2:  pop A, pop B, push A → top = {A, tag=8}
 * Thread 1:  CAS({A,5}, {B,6}) → 실패! (tag 불일치)
 *
 * 64비트 태그는 오버플로우까지 수십억 년 필요
 */
```

### CPU 파이프라인과 backoff

경합 시 exponential backoff가 효과적인 이유:

```cpp
// _mm_pause()의 효과
void push_with_smart_backoff(Node* new_node) {
    Node* old_top = top.load(std::memory_order_relaxed);
    int failures = 0;

    do {
        new_node->next = old_top;

        if (top.compare_exchange_weak(old_top, new_node)) {
            return;
        }

        // Backoff with CPU hints
        if (failures < 4) {
            // 매우 짧은 대기: CPU 파이프라인 flush
            for (int i = 0; i < (1 << failures); i++) {
                _mm_pause();  // ~10-20 사이클
            }
        } else {
            // 긴 대기: OS 스케줄러에게 양보
            std::this_thread::yield();
        }

        failures++;
    } while (true);
}

/*
 * _mm_pause() (x86 PAUSE 명령어)의 효과:
 *
 * 1. 파이프라인 지연 삽입 (~10 사이클)
 * 2. 스핀 루프 인식 → 전력 소비 감소
 * 3. Store Buffer flush 방지 → 다른 스레드 양보
 * 4. 하이퍼스레딩 환경에서 동일 코어의 다른 스레드에게 리소스 양보
 */
```

---

## 🔍 상세 분석

### Memory Ordering 선택

| 연산 | Memory Order | 이유 |
|------|--------------|------|
| `push` load | `relaxed` | 최신 값이 아니어도 CAS에서 재확인됨 |
| `push` CAS (성공) | `release` | 새 노드 데이터가 다른 스레드에 보이도록 |
| `push` CAS (실패) | `relaxed` | 실패 시 재시도하므로 동기화 불필요 |
| `pop` load | `acquire` | 노드 데이터를 읽기 전에 동기화 필요 |
| `pop` CAS (성공) | `release` | top 변경을 다른 스레드에 보이도록 |
| `pop` CAS (실패) | `acquire` | 재시도를 위해 최신 값 필요 |

### ABA 문제

Treiber Stack의 가장 큰 문제는 **ABA Problem**입니다:

```
초기 상태: top → A → B → C

Thread 1: Pop() 시작
1. old_top = A 읽음
2. next = B 읽음
3. (Context switch로 일시 중지)

Thread 2:
1. Pop() → A 제거, top = B
2. Pop() → B 제거, top = C
3. Push(A) → A 재사용, top = A

Thread 1: 재개
1. old_top (A)와 현재 top (A)이 같음
2. CAS 성공! top = B로 설정
3. 문제: B는 이미 삭제되었음 → Dangling pointer!
```

**해결 방법**:
1. **Tagged Pointers**: 포인터에 버전 번호 추가
2. **Hazard Pointers**: 사용 중인 포인터 보호
3. **Epoch-based Reclamation**: 세대 기반 메모리 회수

---

## 📊 성능 특성

| 특성 | 값 |
|------|-----|
| **Push 시간 복잡도** | O(1) 평균, 재시도 시 증가 |
| **Pop 시간 복잡도** | O(1) 평균, 재시도 시 증가 |
| **공간 복잡도** | O(n) (n = 요소 개수) |
| **Progress Guarantee** | Lock-Free (Wait-Free 아님) |
| **확장성** | 우수 (락 없음) |

### 경합 상황 (Contention)

- **저경합**: 매우 빠름 (락 기반보다 우수)
- **고경합**: CAS 재시도 증가로 성능 저하 가능
  - **Backoff 전략** 사용 권장 (지수 백오프)

```cpp
void push_with_backoff(const T& value) {
    Node* new_node = new Node(value);
    Node* old_top = top.load(std::memory_order_relaxed);

    int backoff = 1;
    do {
        new_node->next = old_top;

        if (top.compare_exchange_weak(old_top, new_node,
                                       std::memory_order_release,
                                       std::memory_order_relaxed)) {
            return;
        }

        // 실패 시 백오프
        for (int i = 0; i < backoff; i++) {
            _mm_pause();  // CPU 힌트
        }
        backoff = std::min(backoff * 2, 1024);  // 지수 증가
    } while (true);
}
```

---

## ⚠️ 주의사항과 한계

### 1. 메모리 회수 문제
```cpp
// ❌ 위험: 다른 스레드가 아직 old_top을 읽고 있을 수 있음
delete old_top;
```

**안전한 대안**:
- Hazard Pointers
- Epoch-based Reclamation (RCU)
- Reference Counting (성능 저하)

### 2. ABA 문제
포인터 재사용으로 인한 버그 가능

### 3. Starvation 가능
Lock-Free이지만 Wait-Free는 아님 → 특정 스레드가 계속 실패할 수 있음

### 4. 메모리 순서 버그
잘못된 memory_order 사용 시 미묘한 버그 발생

---

## 🎯 사용 사례

### 적합한 경우
- 높은 동시성 환경
- 락 경합이 심한 시나리오
- 실시간성이 중요한 시스템 (Deadlock 회피)

### 부적합한 경우
- 단일 스레드 또는 저경합 환경 (오버헤드만 증가)
- 복잡한 데이터 구조 (Lock-Free 구현이 너무 어려움)
- 메모리 회수가 복잡한 환경

---

## 🔗 관련 문서

- [CAS 연산](./01-cas-operation.md) - CAS의 기초
- [Memory Ordering](./02-memory-ordering.md) - 메모리 모델 이해
- [ABA Problem](./07-aba-problem.md) - ABA 문제 상세 해결
- [Hazard Pointers](./08-hazard-pointers.md) - 안전한 메모리 회수

---

## 📚 참고 자료

- Treiber, R. K. (1986). "Systems Programming: Coping with Parallelism"
- Herlihy & Shavit. "The Art of Multiprocessor Programming"
- Anthony Williams. "C++ Concurrency in Action"

---

*Treiber Stack은 Lock-Free 프로그래밍의 "Hello World"입니다. 간단하지만 핵심 개념을 모두 담고 있습니다.*
