# Wait-Free Programming

## 📌 개요

**Wait-Free 프로그래밍**은 동시성 알고리즘의 가장 강력한 진전 보장(Progress Guarantee)을 제공합니다. Lock-Free가 "시스템 전체의 진전"을 보장한다면, Wait-Free는 **"모든 개별 스레드의 진전"**을 보장합니다.

Wait-Free 알고리즘에서는 모든 스레드가 유한한 단계(Bounded Number of Steps) 내에 작업을 완료합니다. 즉, 어떤 스레드도 무한히 기다리거나 재시도하지 않습니다.

---

## 🎯 핵심 개념

### Progress Guarantees 계층

```
가장 약함
    ↓
┌─────────────────┐
│   Blocking      │ ← Mutex, Semaphore (진전 보장 없음)
├─────────────────┤
│ Obstruction-Free│ ← 혼자 실행되면 진전
├─────────────────┤
│   Lock-Free     │ ← 시스템 전체가 진전 (일부는 Starvation 가능)
├─────────────────┤
│   Wait-Free     │ ← 모든 스레드가 진전 (Starvation 불가)
└─────────────────┘
    ↑
가장 강함
```

### Wait-Free의 정의

알고리즘이 Wait-Free이려면:
1. **Bounded**: 모든 연산이 O(n) 단계 내에 완료 (n = 스레드 수)
2. **Non-blocking**: 어떤 스레드도 블로킹되지 않음
3. **No Starvation**: 모든 스레드가 공정하게 진행

---

## 📊 Lock-Free vs Wait-Free 비교

### Lock-Free의 한계

```cpp
// Lock-Free Counter (CAS 사용)
void increment() {
    int old_value = value.load();
    while (!value.compare_exchange_weak(old_value, old_value + 1)) {
        // ⚠️ 루프가 무한히 반복될 수 있음!
        // 다른 스레드가 계속 방해하면 이 스레드는 Starvation
    }
}
```

**문제**: 운이 나쁜 스레드는 계속 CAS 실패 → 이론적으로 무한 루프

### Wait-Free의 보장

```cpp
// Wait-Free Counter (Fetch-Add 사용)
void increment() {
    value.fetch_add(1, std::memory_order_relaxed);
    // ✅ 단 한 번의 연산으로 항상 완료!
}
```

**보장**: 모든 스레드가 정확히 1단계에 완료 → Starvation 불가능

---

## 🔬 Wait-Free 알고리즘 예제

### 1. Wait-Free Counter (가장 단순)

```
class WaitFreeCounter {
    atomic<int> value = 0

    procedure Increment()
        // O(1) - 단일 atomic 연산
        value.fetch_add(1, memory_order_relaxed)
    end procedure

    procedure Get() -> int
        // O(1) - 단일 atomic 연산
        return value.load(memory_order_relaxed)
    end procedure
}
```

**분석**:
- ✅ 모든 연산이 정확히 1단계에 완료
- ✅ 하드웨어 atomic 명령어 사용
- ✅ Starvation 불가능

### 2. Wait-Free SPSC Queue (Single Producer Single Consumer)

```
class WaitFreeSPSCQueue {
    T[] buffer[SIZE]
    atomic<int> head = 0  // Consumer가 읽는 위치
    atomic<int> tail = 0  // Producer가 쓰는 위치

    procedure Enqueue(value: T)
        // Producer만 tail 수정 → Wait-Free
        current_tail = tail.load(memory_order_relaxed)
        next_tail = (current_tail + 1) % SIZE

        // Full 체크
        if next_tail == head.load(memory_order_acquire) then
            throw QueueFullException

        buffer[current_tail] = value
        tail.store(next_tail, memory_order_release)
    end procedure

    procedure Dequeue() -> T
        // Consumer만 head 수정 → Wait-Free
        current_head = head.load(memory_order_relaxed)

        // Empty 체크
        if current_head == tail.load(memory_order_acquire) then
            throw QueueEmptyException

        value = buffer[current_head]
        head.store((current_head + 1) % SIZE, memory_order_release)
        return value
    end procedure
}
```

**분석**:
- ✅ Producer와 Consumer가 분리된 변수 접근
- ✅ 경합 없음 → 항상 O(1)에 완료
- ✅ Wait-Free

### 3. Wait-Free MPSC Queue (Kogan-Petrank, 2011)

Wait-Free MPMC 큐는 매우 복잡하지만, MPSC(Multiple Producer Single Consumer)는 비교적 단순:

```
class WaitFreeMPSCQueue {
    struct Node {
        T data
        atomic<Node*> next
    }

    atomic<Node*> head
    atomic<Node*> tail
    atomic<int> enqueue_count  // 도움을 위한 카운터

    procedure Enqueue(value: T)
        new_node = allocate Node(value)

        // Phase 1: tail에 노드 추가 (Lock-Free)
        loop max_tries times
            old_tail = tail.load(memory_order_acquire)
            if compare_exchange_weak(old_tail.next, NULL, new_node) then
                compare_exchange_weak(tail, old_tail, new_node)
                return
            end if
            help_enqueue()  // 다른 스레드 도움
        end loop

        // Phase 2: 실패 시 도움 요청 (Wait-Free 보장)
        enqueue_with_help(new_node)
    end procedure

    procedure Dequeue() -> T
        // Single Consumer → Wait-Free
        old_head = head.load(memory_order_acquire)
        if old_head == tail.load() then
            return NULL
        next = old_head.next.load()
        head.store(next, memory_order_release)
        return next.data
    end procedure
}
```

**핵심 아이디어**:
- Enqueue가 실패하면 다른 스레드에게 도움 요청
- 도움 메커니즘으로 Wait-Free 보장

---

## 💻 C++ 구현 예제

### Wait-Free Read-Write Register

```cpp
#include <atomic>

template<typename T>
class WaitFreeRegister {
private:
    std::atomic<T> value;

public:
    WaitFreeRegister(T initial = T{}) : value(initial) {}

    // Wait-Free Write
    void write(T new_value) {
        value.store(new_value, std::memory_order_release);
    }

    // Wait-Free Read
    T read() const {
        return value.load(std::memory_order_acquire);
    }
};
```

### Wait-Free SPSC Queue (Ring Buffer)

```cpp
#include <atomic>
#include <vector>
#include <optional>

template<typename T, size_t Size>
class WaitFreeSPSCQueue {
private:
    std::vector<T> buffer;
    alignas(64) std::atomic<size_t> head{0};  // Consumer
    alignas(64) std::atomic<size_t> tail{0};  // Producer

public:
    WaitFreeSPSCQueue() : buffer(Size) {}

    // Wait-Free Enqueue (Producer만 tail 수정)
    bool enqueue(const T& value) {
        size_t current_tail = tail.load(std::memory_order_relaxed);
        size_t next_tail = (current_tail + 1) % Size;

        // Full 체크
        if (next_tail == head.load(std::memory_order_acquire)) {
            return false;  // Queue Full
        }

        buffer[current_tail] = value;
        tail.store(next_tail, std::memory_order_release);
        return true;
    }

    // Wait-Free Dequeue (Consumer만 head 수정)
    std::optional<T> dequeue() {
        size_t current_head = head.load(std::memory_order_relaxed);

        // Empty 체크
        if (current_head == tail.load(std::memory_order_acquire)) {
            return std::nullopt;  // Queue Empty
        }

        T value = buffer[current_head];
        head.store((current_head + 1) % Size, std::memory_order_release);
        return value;
    }

    bool empty() const {
        return head.load(std::memory_order_relaxed) ==
               tail.load(std::memory_order_acquire);
    }

    bool full() const {
        size_t current_tail = tail.load(std::memory_order_relaxed);
        size_t next_tail = (current_tail + 1) % Size;
        return next_tail == head.load(std::memory_order_acquire);
    }
};
```

---

## 🔍 Wait-Free 알고리즘 설계 원칙

### 1. 하드웨어 Atomic 연산 활용

대부분의 하드웨어 atomic 연산은 Wait-Free:
- `fetch_add` / `fetch_sub`
- `exchange`
- `load` / `store`

```cpp
// ✅ Wait-Free
counter.fetch_add(1);

// ❌ Lock-Free (Wait-Free 아님)
int old = counter.load();
while (!counter.compare_exchange_weak(old, old + 1)) {}
```

### 2. 단일 Writer 패턴

여러 reader + 단일 writer → Wait-Free 구현 쉬움

```cpp
// Writer: 단일 스레드만 write
data.store(new_value, std::memory_order_release);

// Readers: 여러 스레드가 read (항상 Wait-Free)
T value = data.load(std::memory_order_acquire);
```

### 3. 도움 메커니즘 (Helping)

스레드가 실패 시 다른 스레드에게 도움 요청:

```cpp
struct HelpRequest {
    std::atomic<bool> pending{false};
    T data;
};

HelpRequest help_requests[MAX_THREADS];

void enqueue_with_help(T value, int thread_id) {
    help_requests[thread_id].data = value;
    help_requests[thread_id].pending.store(true);

    // 다른 스레드가 도움
    for (int i = 0; i < MAX_THREADS; i++) {
        if (help_requests[i].pending.load()) {
            complete_operation(help_requests[i]);
        }
    }
}
```

### 4. 사전 할당 (Pre-allocation)

메모리 할당은 Wait-Free가 아니므로 미리 할당:

```cpp
// ❌ 동적 할당 → Wait-Free 아님
Node* node = new Node();

// ✅ 사전 할당 풀 사용
Node* node = node_pool.acquire();  // Lock-Free/Wait-Free 풀 필요
```

---

## 📊 Wait-Free의 장단점

### 장점

| 특성 | 설명 |
|------|------|
| **최강의 보장** | 모든 스레드가 진전 |
| **예측 가능한 레이턴시** | O(n) 시간 보장 |
| **실시간성** | Hard Real-Time 시스템에 적합 |
| **No Starvation** | 공정성 보장 |
| **확장성** | 스레드 수 증가에도 일정한 성능 |

### 단점

| 특성 | 설명 |
|------|------|
| **구현 매우 어려움** | Lock-Free보다 훨씬 복잡 |
| **오버헤드 증가** | 도움 메커니즘 등의 추가 비용 |
| **실용성 낮음** | 대부분 이론적, 실제 구현 적음 |
| **디버깅 어려움** | 복잡한 상태 관리 |

---

## 🎯 사용 사례

### Wait-Free가 필수인 경우

1. **Hard Real-Time 시스템**
   - 항공 전자 장비
   - 의료 기기
   - 자동차 제어 시스템

2. **극도로 높은 공정성 요구**
   - 금융 거래 시스템
   - 공정 스케줄링

3. **Starvation이 치명적인 경우**
   - 안전 critical 시스템

### Lock-Free로 충분한 경우

1. **일반 서버 애플리케이션**
   - 웹 서버
   - 데이터베이스

2. **게임 서버**
   - Soft Real-Time (몇 밀리초 지연 허용)

3. **대부분의 경우**
   - Wait-Free의 복잡도가 너무 높음

---

## ⚠️ Wait-Free 구현의 도전 과제

### 1. MPMC는 거의 불가능

Wait-Free MPMC (Multiple Producer Multiple Consumer) 큐는 이론적으로 가능하지만:
- 극도로 복잡
- 성능 오버헤드 큼
- 실용성 낮음

**현실적 대안**: Lock-Free MPMC (Michael-Scott Queue)

### 2. 메모리 할당 문제

`new`/`delete`는 Wait-Free가 아님:

```cpp
// ❌ Wait-Free 아님
Node* node = new Node();  // 메모리 할당자가 락 사용 가능

// ✅ Wait-Free 풀 필요
Node* node = wait_free_pool.acquire();
```

### 3. ABA 문제 해결 어려움

Wait-Free에서도 ABA 문제 발생 → Hazard Pointers 등 필요

---

## 🔗 Wait-Free 알고리즘 목록

### 실용적인 Wait-Free 알고리즘

| 알고리즘 | 복잡도 | 사용 사례 |
|----------|--------|-----------|
| Fetch-Add Counter | O(1) | 통계, 카운팅 |
| Read-Write Register | O(1) | 단순 공유 변수 |
| SPSC Queue | O(1) | Producer-Consumer |
| Snapshot | O(n²) | 일관된 상태 읽기 |

### 이론적인 Wait-Free 알고리즘

| 알고리즘 | 복잡도 | 비고 |
|----------|--------|------|
| Universal Construction | O(n²) | 모든 자료구조 Wait-Free로 변환 (이론적) |
| Kogan-Petrank MPSC Queue | O(n) | 복잡하지만 실용적 |
| Wait-Free Stack | O(n) | 매우 복잡, 오버헤드 큼 |

---

## 📚 고급 주제

### Universal Construction

Herlihy의 Universal Construction: 모든 순차적 자료구조를 Wait-Free로 변환

**아이디어**:
1. 연산을 명령어 로그에 추가
2. 모든 스레드가 로그 재생
3. Consensus 프로토콜로 순서 결정

**문제**: 성능 오버헤드가 너무 커서 실용성 낮음

### Wait-Free Snapshot

여러 atomic 변수의 일관된 스냅샷 읽기:

```cpp
// 목표: 모든 카운터의 일관된 값 읽기
std::atomic<int> counters[N];

// Wait-Free Snapshot (Afek et al., 1993)
struct Snapshot {
    int values[N];
    int version;
};

Snapshot read_snapshot() {
    // O(n²) 복잡도의 Wait-Free 알고리즘
    // ... (구현 복잡)
}
```

---

## 🔗 관련 문서

- [Lock-Free Counter](./05-lock-free-counter.md) - Wait-Free 예제
- [Lock-Free Stack](./03-lock-free-stack.md) - Lock-Free와 비교
- [Lock-Free Queue](./04-lock-free-queue.md) - Lock-Free와 비교

---

## 📚 참고 자료

### 논문
- Herlihy (1991). "Wait-Free Synchronization"
- Kogan & Petrank (2011). "Wait-Free Queues With Multiple Enqueuers and Dequeuers"
- Afek et al. (1993). "Atomic Snapshots of Shared Memory"

### 서적
- Herlihy & Shavit. "The Art of Multiprocessor Programming" - Chapter 3
- Maurice Herlihy. "Wait-Free Synchronization" (ACM TOPLAS 1991)

### 온라인 자료
- [Preshing on Programming - Lock-Free vs Wait-Free](https://preshing.com/20120612/an-introduction-to-lock-free-programming/)

---

## 💡 결론

### Wait-Free를 사용해야 하는 경우
- ✅ Hard Real-Time 시스템
- ✅ Starvation이 절대 불가능해야 하는 경우
- ✅ 극도로 높은 공정성 필요

### Lock-Free로 충분한 경우
- ✅ 대부분의 서버 애플리케이션
- ✅ 게임 서버 (Soft Real-Time)
- ✅ 일반적인 동시성 문제

**실용적 조언**: 단순한 연산(Counter, Register)은 Wait-Free, 복잡한 자료구조는 Lock-Free 사용

---

*Wait-Free는 이론적으로 가장 강력하지만, 실전에서는 Lock-Free로 충분한 경우가 대부분입니다. 복잡도와 성능의 트레이드오프를 신중히 고려하세요.*
