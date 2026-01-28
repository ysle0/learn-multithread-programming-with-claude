# Lock-Free Queue 변형: SPSC, MPSC, MPMC

## 목차
1. [개요](#개요)
2. [SPSC Queue](#spsc-queue)
3. [MPSC Queue](#mpsc-queue)
4. [MPMC Queue](#mpmc-queue)
5. [성능 비교](#성능-비교)
6. [선택 가이드](#선택-가이드)
7. [요약](#요약)

---

## 개요

Lock-Free Queue는 Producer(생산자)와 Consumer(소비자)의 수에 따라 여러 변형이 있습니다. 각 변형은 특정 사용 사례에 최적화되어 있습니다.

### Queue 유형 분류

```
┌─────────────────────────────────────────────────────────────────┐
│                    Queue 유형 분류                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer 수 × Consumer 수 조합:                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    │ Single Consumer │ Multi Consumer   │   │
│  │────────────────────│─────────────────│──────────────────│   │
│  │ Single Producer    │     SPSC        │      SPMC        │   │
│  │────────────────────│─────────────────│──────────────────│   │
│  │ Multi Producer     │     MPSC        │      MPMC        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  복잡도 및 성능 (일반적):                                        │
│                                                                 │
│  SPSC ─────────────────────────────────────────────► MPMC      │
│  간단              복잡도 증가              복잡                 │
│  빠름              성능 저하               느림                 │
│  Wait-Free 가능    Lock-Free              Lock-Free            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 사용 사례

| 유형 | 사용 사례 |
|------|----------|
| **SPSC** | 오디오 처리, 센서 데이터, 단일 파이프라인 |
| **SPMC** | 이벤트 브로드캐스트, 작업 분배 |
| **MPSC** | 로깅 시스템, 이벤트 수집, Actor 메시지 큐 |
| **MPMC** | 스레드 풀 작업 큐, 범용 메시지 큐 |

---

## SPSC Queue

**Single Producer, Single Consumer** - 가장 단순하고 빠른 형태입니다.

### 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPSC Queue 특징                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                        ┌──────────────┐      │
│  │   Producer   │ ──────────────────────►│   Consumer   │      │
│  │   (1개만)    │         Queue          │   (1개만)    │      │
│  └──────────────┘                        └──────────────┘      │
│                                                                 │
│  장점:                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - CAS 불필요 (단순 load/store)                           │   │
│  │ - Wait-Free 구현 가능                                    │   │
│  │ - 캐시 효율적                                            │   │
│  │ - 가장 낮은 지연시간                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  단점:                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 하나의 Producer, 하나의 Consumer로 제한               │   │
│  │ - 유연성 낮음                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Ring Buffer 구현

```c
#include <stdatomic.h>
#include <stdbool.h>
#include <stddef.h>

#define CACHE_LINE_SIZE 64

// 캐시 라인 분리로 False Sharing 방지
typedef struct {
    alignas(CACHE_LINE_SIZE) atomic_size_t head;  // Consumer가 읽는 위치
    alignas(CACHE_LINE_SIZE) atomic_size_t tail;  // Producer가 쓰는 위치
    size_t capacity;
    void** buffer;
} SPSCQueue;

void spsc_init(SPSCQueue* q, size_t capacity) {
    // capacity는 2의 제곱이어야 함 (모듈로 연산 최적화)
    q->capacity = capacity;
    q->buffer = malloc(capacity * sizeof(void*));
    atomic_store(&q->head, 0);
    atomic_store(&q->tail, 0);
}

// Producer: 데이터 추가
bool spsc_push(SPSCQueue* q, void* item) {
    size_t tail = atomic_load_explicit(&q->tail, memory_order_relaxed);
    size_t next_tail = (tail + 1) & (q->capacity - 1);  // % 대신 &

    // 큐가 가득 찼는지 확인 (head와 비교)
    size_t head = atomic_load_explicit(&q->head, memory_order_acquire);
    if (next_tail == head) {
        return false;  // Full
    }

    q->buffer[tail] = item;

    // tail 업데이트 (Consumer에게 보이도록 release)
    atomic_store_explicit(&q->tail, next_tail, memory_order_release);
    return true;
}

// Consumer: 데이터 꺼내기
bool spsc_pop(SPSCQueue* q, void** item) {
    size_t head = atomic_load_explicit(&q->head, memory_order_relaxed);

    // 큐가 비었는지 확인 (tail과 비교)
    size_t tail = atomic_load_explicit(&q->tail, memory_order_acquire);
    if (head == tail) {
        return false;  // Empty
    }

    *item = q->buffer[head];

    // head 업데이트 (Producer에게 보이도록 release)
    size_t next_head = (head + 1) & (q->capacity - 1);
    atomic_store_explicit(&q->head, next_head, memory_order_release);
    return true;
}
```

### 최적화된 SPSC (Batch & Local Cache)

```c
typedef struct {
    alignas(CACHE_LINE_SIZE) size_t head;
    alignas(CACHE_LINE_SIZE) size_t tail;
    alignas(CACHE_LINE_SIZE) size_t cached_head;  // Producer 로컬 캐시
    alignas(CACHE_LINE_SIZE) size_t cached_tail;  // Consumer 로컬 캐시
    size_t capacity;
    void** buffer;
} OptimizedSPSCQueue;

// Producer: 캐시된 head 사용으로 atomic 읽기 감소
bool opt_spsc_push(OptimizedSPSCQueue* q, void* item) {
    size_t tail = q->tail;
    size_t next_tail = (tail + 1) & (q->capacity - 1);

    // 캐시된 head로 먼저 확인
    if (next_tail == q->cached_head) {
        // 캐시 미스: 실제 head 읽기
        q->cached_head = atomic_load_explicit(
            (atomic_size_t*)&q->head, memory_order_acquire);
        if (next_tail == q->cached_head) {
            return false;  // 정말로 Full
        }
    }

    q->buffer[tail] = item;
    atomic_store_explicit((atomic_size_t*)&q->tail,
                          next_tail, memory_order_release);
    q->tail = next_tail;
    return true;
}

// Batch push: 여러 아이템을 한 번에
size_t opt_spsc_push_batch(OptimizedSPSCQueue* q, void** items, size_t count) {
    size_t tail = q->tail;
    size_t head = atomic_load_explicit(
        (atomic_size_t*)&q->head, memory_order_acquire);

    size_t available = (head - tail - 1) & (q->capacity - 1);
    size_t to_push = (count < available) ? count : available;

    for (size_t i = 0; i < to_push; i++) {
        q->buffer[(tail + i) & (q->capacity - 1)] = items[i];
    }

    atomic_store_explicit((atomic_size_t*)&q->tail,
                          (tail + to_push) & (q->capacity - 1),
                          memory_order_release);
    return to_push;
}
```

---

## MPSC Queue

**Multiple Producer, Single Consumer** - 여러 Producer가 하나의 Consumer에게 전달합니다.

### 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                    MPSC Queue 특징                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │  Producer 1  │──┐                                            │
│  └──────────────┘  │                                            │
│  ┌──────────────┐  │      ┌─────────┐      ┌──────────────┐    │
│  │  Producer 2  │──┼─────►│  Queue  │─────►│   Consumer   │    │
│  └──────────────┘  │      └─────────┘      │   (1개만)    │    │
│  ┌──────────────┐  │                       └──────────────┘    │
│  │  Producer N  │──┘                                            │
│  └──────────────┘                                               │
│                                                                 │
│  사용 사례:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 로깅 시스템 (여러 스레드 → 로그 작성자)                │   │
│  │ - 이벤트 수집기                                          │   │
│  │ - Actor 모델의 메시지 큐                                 │   │
│  │ - Metrics 집계                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Intrusive Linked List 구현 (Dmitry Vyukov)

```c
#include <stdatomic.h>
#include <stddef.h>

typedef struct MPSCNode {
    _Atomic(struct MPSCNode*) next;
} MPSCNode;

typedef struct {
    alignas(64) _Atomic(MPSCNode*) head;  // Producer들이 push하는 곳
    alignas(64) MPSCNode* tail;           // Consumer가 pop하는 곳
    MPSCNode stub;                        // Dummy 노드
} MPSCQueue;

void mpsc_init(MPSCQueue* q) {
    atomic_store(&q->stub.next, NULL);
    atomic_store(&q->head, &q->stub);
    q->tail = &q->stub;
}

// Producer: Lock-Free push (CAS 루프)
void mpsc_push(MPSCQueue* q, MPSCNode* node) {
    atomic_store_explicit(&node->next, NULL, memory_order_relaxed);

    // head를 새 노드로 교체하고 이전 head 획득
    MPSCNode* prev = atomic_exchange_explicit(&q->head, node,
                                               memory_order_acq_rel);

    // 이전 노드의 next를 새 노드로 연결
    atomic_store_explicit(&prev->next, node, memory_order_release);
}

// Consumer: 단일 Consumer이므로 CAS 불필요
MPSCNode* mpsc_pop(MPSCQueue* q) {
    MPSCNode* tail = q->tail;
    MPSCNode* next = atomic_load_explicit(&tail->next, memory_order_acquire);

    if (tail == &q->stub) {
        // stub 노드 건너뛰기
        if (next == NULL) {
            return NULL;  // Empty
        }
        q->tail = next;
        tail = next;
        next = atomic_load_explicit(&tail->next, memory_order_acquire);
    }

    if (next != NULL) {
        q->tail = next;
        return tail;
    }

    // head와 tail이 같은지 확인 (마지막 노드)
    MPSCNode* head = atomic_load_explicit(&q->head, memory_order_acquire);
    if (tail != head) {
        // Producer가 push 중 (next가 아직 NULL)
        return NULL;  // 잠시 후 재시도
    }

    // stub 재삽입하여 큐 유지
    mpsc_push(q, &q->stub);

    next = atomic_load_explicit(&tail->next, memory_order_acquire);
    if (next != NULL) {
        q->tail = next;
        return tail;
    }

    return NULL;
}
```

### Bounded MPSC Queue (Ring Buffer)

```c
typedef struct {
    alignas(64) atomic_size_t head;
    alignas(64) atomic_size_t tail;
    size_t mask;
    _Atomic(void*)* buffer;
} BoundedMPSCQueue;

void bounded_mpsc_init(BoundedMPSCQueue* q, size_t capacity) {
    q->mask = capacity - 1;  // capacity는 2의 제곱
    q->buffer = calloc(capacity, sizeof(_Atomic(void*)));
    atomic_store(&q->head, 0);
    atomic_store(&q->tail, 0);
}

// Producer: CAS로 tail 경쟁
bool bounded_mpsc_push(BoundedMPSCQueue* q, void* item) {
    size_t tail;
    size_t next_tail;

    do {
        tail = atomic_load_explicit(&q->tail, memory_order_relaxed);
        next_tail = (tail + 1) & q->mask;

        // Full 확인
        size_t head = atomic_load_explicit(&q->head, memory_order_acquire);
        if (next_tail == head) {
            return false;
        }
    } while (!atomic_compare_exchange_weak_explicit(
        &q->tail, &tail, next_tail,
        memory_order_acq_rel, memory_order_relaxed));

    // 슬롯 확보됨, 데이터 저장
    atomic_store_explicit(&q->buffer[tail], item, memory_order_release);
    return true;
}

// Consumer: 단일이므로 CAS 불필요
bool bounded_mpsc_pop(BoundedMPSCQueue* q, void** item) {
    size_t head = atomic_load_explicit(&q->head, memory_order_relaxed);
    size_t tail = atomic_load_explicit(&q->tail, memory_order_acquire);

    if (head == tail) {
        return false;  // Empty
    }

    // 데이터가 준비될 때까지 대기 (Producer가 store 완료)
    void* data;
    while ((data = atomic_load_explicit(&q->buffer[head],
                                         memory_order_acquire)) == NULL) {
        // spin
    }

    *item = data;
    atomic_store_explicit(&q->buffer[head], NULL, memory_order_relaxed);
    atomic_store_explicit(&q->head, (head + 1) & q->mask, memory_order_release);
    return true;
}
```

---

## MPMC Queue

**Multiple Producer, Multiple Consumer** - 가장 범용적이지만 가장 복잡합니다.

### 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                    MPMC Queue 특징                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                        ┌──────────────┐      │
│  │  Producer 1  │──┐                  ┌──│  Consumer 1  │      │
│  └──────────────┘  │                  │  └──────────────┘      │
│  ┌──────────────┐  │    ┌─────────┐   │  ┌──────────────┐      │
│  │  Producer 2  │──┼───►│  Queue  │───┼──│  Consumer 2  │      │
│  └──────────────┘  │    └─────────┘   │  └──────────────┘      │
│  ┌──────────────┐  │                  │  ┌──────────────┐      │
│  │  Producer N  │──┘                  └──│  Consumer M  │      │
│  └──────────────┘                        └──────────────┘      │
│                                                                 │
│  복잡성:                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - Producer 간 경쟁                                       │   │
│  │ - Consumer 간 경쟁                                       │   │
│  │ - Producer-Consumer 동기화                               │   │
│  │ - ABA 문제 고려                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Bounded MPMC Queue (Dmitry Vyukov)

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct {
    void* data;
    atomic_size_t sequence;
} MPMCCell;

typedef struct {
    alignas(64) atomic_size_t enqueue_pos;
    alignas(64) atomic_size_t dequeue_pos;
    size_t mask;
    MPMCCell* buffer;
} MPMCQueue;

void mpmc_init(MPMCQueue* q, size_t capacity) {
    q->mask = capacity - 1;
    q->buffer = aligned_alloc(64, capacity * sizeof(MPMCCell));

    for (size_t i = 0; i < capacity; i++) {
        atomic_store(&q->buffer[i].sequence, i);
    }

    atomic_store(&q->enqueue_pos, 0);
    atomic_store(&q->dequeue_pos, 0);
}

// Producer
bool mpmc_push(MPMCQueue* q, void* item) {
    MPMCCell* cell;
    size_t pos;

    while (true) {
        pos = atomic_load_explicit(&q->enqueue_pos, memory_order_relaxed);
        cell = &q->buffer[pos & q->mask];
        size_t seq = atomic_load_explicit(&cell->sequence, memory_order_acquire);
        intptr_t diff = (intptr_t)seq - (intptr_t)pos;

        if (diff == 0) {
            // 슬롯 사용 가능
            if (atomic_compare_exchange_weak_explicit(
                    &q->enqueue_pos, &pos, pos + 1,
                    memory_order_relaxed, memory_order_relaxed)) {
                break;
            }
        } else if (diff < 0) {
            // 큐가 가득 참
            return false;
        }
        // diff > 0: 다른 Producer가 진행 중, 재시도
    }

    cell->data = item;
    atomic_store_explicit(&cell->sequence, pos + 1, memory_order_release);
    return true;
}

// Consumer
bool mpmc_pop(MPMCQueue* q, void** item) {
    MPMCCell* cell;
    size_t pos;

    while (true) {
        pos = atomic_load_explicit(&q->dequeue_pos, memory_order_relaxed);
        cell = &q->buffer[pos & q->mask];
        size_t seq = atomic_load_explicit(&cell->sequence, memory_order_acquire);
        intptr_t diff = (intptr_t)seq - (intptr_t)(pos + 1);

        if (diff == 0) {
            // 데이터 사용 가능
            if (atomic_compare_exchange_weak_explicit(
                    &q->dequeue_pos, &pos, pos + 1,
                    memory_order_relaxed, memory_order_relaxed)) {
                break;
            }
        } else if (diff < 0) {
            // 큐가 비어있음
            return false;
        }
        // diff > 0: 다른 Consumer가 진행 중, 재시도
    }

    *item = cell->data;
    atomic_store_explicit(&cell->sequence, pos + q->mask + 1, memory_order_release);
    return true;
}
```

### Sequence 번호의 역할

```
┌─────────────────────────────────────────────────────────────────┐
│                  MPMC Sequence 동작 원리                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  초기 상태 (capacity = 4):                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Index │  0  │  1  │  2  │  3  │                         │   │
│  │ Seq   │  0  │  1  │  2  │  3  │                         │   │
│  │ Data  │ null│ null│ null│ null│                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│  enqueue_pos = 0, dequeue_pos = 0                               │
│                                                                 │
│  Push 후:                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Index │  0  │  1  │  2  │  3  │                         │   │
│  │ Seq   │  1  │  1  │  2  │  3  │  ← seq = pos + 1       │   │
│  │ Data  │  A  │ null│ null│ null│                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│  enqueue_pos = 1                                                │
│                                                                 │
│  Sequence 의미:                                                  │
│  - seq == pos: Push 가능 (슬롯 비어있음)                         │
│  - seq == pos + 1: Pop 가능 (데이터 있음)                        │
│  - seq < pos: 큐 가득 참                                        │
│  - seq < pos + 1: 큐 비어있음                                   │
│                                                                 │
│  Pop 후:                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Index │  0  │  1  │  2  │  3  │                         │   │
│  │ Seq   │  4  │  1  │  2  │  3  │  ← seq = pos + mask + 1│   │
│  │ Data  │ null│ null│ null│ null│                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│  dequeue_pos = 1                                                │
│  다음 라운드에서 index 0은 seq=4일 때 Push 가능                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Michael-Scott Queue (Unbounded MPMC)

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdbool.h>

typedef struct MSNode {
    void* data;
    _Atomic(struct MSNode*) next;
} MSNode;

typedef struct {
    alignas(64) _Atomic(MSNode*) head;
    alignas(64) _Atomic(MSNode*) tail;
} MSQueue;

void ms_init(MSQueue* q) {
    MSNode* dummy = malloc(sizeof(MSNode));
    dummy->data = NULL;
    atomic_store(&dummy->next, NULL);
    atomic_store(&q->head, dummy);
    atomic_store(&q->tail, dummy);
}

// Push (Lock-Free)
void ms_push(MSQueue* q, void* item) {
    MSNode* node = malloc(sizeof(MSNode));
    node->data = item;
    atomic_store(&node->next, NULL);

    while (true) {
        MSNode* tail = atomic_load(&q->tail);
        MSNode* next = atomic_load(&tail->next);

        if (tail == atomic_load(&q->tail)) {
            if (next == NULL) {
                // tail이 마지막 노드
                if (atomic_compare_exchange_weak(&tail->next, &next, node)) {
                    // 성공, tail 업데이트 시도 (실패해도 OK)
                    atomic_compare_exchange_weak(&q->tail, &tail, node);
                    return;
                }
            } else {
                // tail이 뒤쳐져 있음, 따라잡기
                atomic_compare_exchange_weak(&q->tail, &tail, next);
            }
        }
    }
}

// Pop (Lock-Free)
void* ms_pop(MSQueue* q) {
    while (true) {
        MSNode* head = atomic_load(&q->head);
        MSNode* tail = atomic_load(&q->tail);
        MSNode* next = atomic_load(&head->next);

        if (head == atomic_load(&q->head)) {
            if (head == tail) {
                if (next == NULL) {
                    return NULL;  // Empty
                }
                // tail이 뒤쳐져 있음
                atomic_compare_exchange_weak(&q->tail, &tail, next);
            } else {
                void* data = next->data;
                if (atomic_compare_exchange_weak(&q->head, &head, next)) {
                    // 주의: head는 이전 dummy 노드, free 필요
                    // Hazard Pointer 또는 다른 메모리 회수 기법 필요
                    return data;
                }
            }
        }
    }
}
```

---

## 성능 비교

### 벤치마크 결과 (일반적인 경향)

```
┌─────────────────────────────────────────────────────────────────┐
│               Queue 성능 비교 (상대적)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  처리량 (ops/sec):                                               │
│                                                                 │
│  SPSC:  ████████████████████████████████████████  100M ops/sec │
│  MPSC:  █████████████████████████████  60M ops/sec             │
│  SPMC:  ████████████████████████  50M ops/sec                  │
│  MPMC:  ██████████████████  40M ops/sec                        │
│                                                                 │
│  지연시간 (latency):                                             │
│                                                                 │
│  SPSC:  ██  ~10-20 ns                                          │
│  MPSC:  ████  ~30-50 ns                                        │
│  SPMC:  █████  ~40-60 ns                                       │
│  MPMC:  ███████  ~50-100 ns                                    │
│                                                                 │
│  참고: 실제 성능은 구현, 하드웨어, 경합 수준에 따라 다름          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 경합 수준별 성능

```
┌─────────────────────────────────────────────────────────────────┐
│             경합 수준에 따른 성능 변화                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  낮은 경합 (스레드 < 코어):                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - MPMC도 좋은 성능                                       │   │
│  │ - CAS 성공률 높음                                        │   │
│  │ - 캐시 효율 좋음                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  높은 경합 (스레드 >> 코어):                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - MPMC 성능 급격히 저하                                  │   │
│  │ - CAS 실패 및 재시도 증가                                │   │
│  │ - 캐시 라인 핑퐁                                         │   │
│  │ - SPSC/MPSC가 상대적으로 유리                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│                    Queue 선택 플로우차트                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Producer가 1개인가?                                             │
│      │                                                          │
│      ├── Yes ──► Consumer가 1개인가?                           │
│      │               │                                          │
│      │               ├── Yes ──► SPSC (최고 성능)              │
│      │               │                                          │
│      │               └── No ──► SPMC                           │
│      │                                                          │
│      └── No ──► Consumer가 1개인가?                            │
│                      │                                          │
│                      ├── Yes ──► MPSC                          │
│                      │           (로깅, Actor 메시지 등)        │
│                      │                                          │
│                      └── No ──► MPMC                           │
│                                  (범용, Thread Pool)            │
│                                                                 │
│  추가 고려사항:                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - Bounded vs Unbounded                                   │   │
│  │   Bounded: 메모리 제한, 배압(backpressure) 가능          │   │
│  │   Unbounded: 메모리 무제한 증가 가능, OOM 위험           │   │
│  │                                                          │   │
│  │ - Wait-Free vs Lock-Free                                │   │
│  │   SPSC는 Wait-Free 가능 (지연시간 보장)                  │   │
│  │   나머지는 Lock-Free (전체 진행 보장)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 요약

### Queue 유형 비교

| 유형 | Producer | Consumer | 복잡도 | 성능 | 사용 사례 |
|------|----------|----------|--------|------|----------|
| **SPSC** | 1 | 1 | 낮음 | 최고 | 파이프라인, 오디오 |
| **SPMC** | 1 | N | 중간 | 좋음 | 브로드캐스트 |
| **MPSC** | N | 1 | 중간 | 좋음 | 로깅, Actor |
| **MPMC** | N | M | 높음 | 보통 | Thread Pool |

### 핵심 구현 기법

| 기법 | 적용 대상 | 효과 |
|------|----------|------|
| Ring Buffer | 모든 유형 | 메모리 효율, 캐시 친화적 |
| Sequence 번호 | MPMC | ABA 방지, 상태 추적 |
| 캐시 라인 분리 | 모든 유형 | False Sharing 방지 |
| Batch 연산 | 모든 유형 | 처리량 향상 |

---

## 🔧 내부 메커니즘

### SPSC Ring Buffer의 캐시 동작

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SPSC 메모리 접근 패턴                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Producer (CPU 0)                Consumer (CPU 1)                   │
│  ───────────────                 ────────────────                   │
│                                                                     │
│  L1 Cache:                       L1 Cache:                          │
│  ┌─────────────────┐             ┌─────────────────┐                │
│  │ tail: Exclusive │             │ head: Exclusive │                │
│  │ head: Shared    │             │ tail: Shared    │                │
│  │ buffer[tail]:M  │             │ buffer[head]:S  │                │
│  └─────────────────┘             └─────────────────┘                │
│                                                                     │
│  핵심 최적화:                                                        │
│  1. head와 tail이 다른 캐시 라인 → False Sharing 없음                │
│  2. Producer는 tail만 수정, Consumer는 head만 수정                   │
│  3. buffer 접근은 순차적 → 프리페치 효과적                           │
│                                                                     │
│  메모리 대역폭:                                                      │
│  - Producer: 2 cache line read (tail, head) + 1 write (buffer)      │
│  - Consumer: 2 cache line read (head, tail) + 1 write (buffer)      │
│  - 캐시된 인덱스로 대부분 L1 hit                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Vyukov MPMC Queue Sequence 상세 분석

```cpp
// Sequence 번호가 왜 필요한가?

// 시나리오: 2개 Producer (P1, P2), capacity=4
// 초기: seq = [0, 1, 2, 3], enqueue_pos = 0

// P1: pos=0 획득, CAS 성공
// P2: pos=1 획득, CAS 성공

// 만약 seq 없이 단순히 enqueue_pos만 사용한다면:
// P1이 느려서 buffer[0]에 아직 저장 안 함
// Consumer가 dequeue_pos=0 에서 읽으려 함
// → 아직 데이터 없음! (race condition)

// Sequence로 해결:
// P1: seq[0]=0 → push 가능, 데이터 저장 후 seq[0]=1로 변경
// Consumer: seq[0]=1 → pop 가능 (데이터 준비됨 확인)
// 만약 seq[0]=0이면 아직 준비 안 됨 → 대기

// Sequence 상태 머신:
// ┌─────────────────────────────────────────────────────┐
// │  seq = pos     : Producer가 이 슬롯에 push 가능      │
// │  seq = pos + 1 : Consumer가 이 슬롯에서 pop 가능     │
// │  seq > pos + 1 : 다른 Producer/Consumer가 진행 중    │
// │  seq < pos     : 큐 가득 참 (wraparound)            │
// │  seq < pos + 1 : 큐 비어있음                         │
// └─────────────────────────────────────────────────────┘
```

**Wraparound 처리**:
```
┌─────────────────────────────────────────────────────────────────────┐
│                    Sequence Wraparound                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  capacity = 4, mask = 3                                             │
│                                                                     │
│  라운드 0:                                                           │
│  pos: 0, 1, 2, 3                                                    │
│  seq: 0, 1, 2, 3 → push 가능                                        │
│                                                                     │
│  push 4회 후:                                                        │
│  seq: 1, 2, 3, 4  (각 seq = pos + 1)                                │
│  enqueue_pos = 4                                                    │
│                                                                     │
│  pop 4회 후:                                                         │
│  seq: 4, 5, 6, 7  (각 seq = pos + mask + 1)                         │
│  dequeue_pos = 4                                                    │
│                                                                     │
│  라운드 1 (pos 4~7이 슬롯 0~3 매핑):                                  │
│  pos=4 → slot=0, seq[0]=4 == pos → push 가능                        │
│  pos=5 → slot=1, seq[1]=5 == pos → push 가능                        │
│  ...                                                                │
│                                                                     │
│  결론: seq가 pos와 동기화되어 ABA 문제 자연스럽게 해결                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### MPSC Queue의 두 단계 Push

```cpp
// Dmitry Vyukov's MPSC: 왜 두 단계가 필요한가?

void mpsc_push(MPSCQueue* q, Node* node) {
    node->next = NULL;

    // 단계 1: head를 새 노드로 원자적 교환
    Node* prev = atomic_exchange(&q->head, node);

    // ⚠️ 여기서 선점되면?
    // prev->next가 아직 NULL
    // Consumer가 prev까지 왔는데 next가 NULL → 큐가 비어보임!

    // 단계 2: 이전 노드의 next를 새 노드로 연결
    atomic_store(&prev->next, node);
}

// 해결: Consumer가 이 "틈"을 감지하고 대기
Node* mpsc_pop(MPSCQueue* q) {
    Node* tail = q->tail;
    Node* next = atomic_load(&tail->next);

    if (next == NULL) {
        // 두 가지 가능성:
        // 1. 큐가 정말 비어있음
        // 2. Producer가 exchange 후 store 전

        // head와 비교로 구분
        if (tail == atomic_load(&q->head)) {
            return NULL;  // 정말 비어있음
        }
        // Producer가 진행 중 → 잠시 후 재시도
        return NULL;  // 또는 spin wait
    }
    // ...
}
```

### 캐시 라인 경합 측정

```bash
# perf로 캐시 경합 분석
$ perf c2c record ./queue_benchmark
$ perf c2c report

# 결과 예시:
# =================================================
#            Shared Data Cache Line Table
# =================================================
# HITM  Tot Hitm  Lcl Hitm  Rmt Hitm   Samples  Symbol
# 45.2%    1.2K     0.3K      0.9K      10.5K  enqueue_pos  ← 경합!
# 32.1%    0.8K     0.2K      0.6K       8.2K  dequeue_pos  ← 경합!
#  5.3%    0.1K       -        0.1K       1.5K  sequence[0]

# HITM = Hit-In-Modified (다른 캐시의 Modified 라인 접근)
# Rmt Hitm = 원격 소켓 캐시 접근 (NUMA에서 특히 비쌈)

# 해결: alignas(64)로 각각 다른 캐시 라인에 배치
```

### 지연시간 분석

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Queue 연산별 지연시간 분해                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  SPSC Push (최적 경로):                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. tail load (L1 hit)           ~1 cycle                    │    │
│  │ 2. head load (L1/L2 hit)        ~3-10 cycles                │    │
│  │ 3. buffer[tail] store           ~1 cycle                    │    │
│  │ 4. tail store (release)         ~1 cycle                    │    │
│  │ ─────────────────────────────────                           │    │
│  │ 총: ~6-15 cycles (~2-5 ns @ 3GHz)                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  MPMC Push (경합 시):                                                │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 1. enqueue_pos load             ~1 cycle                    │    │
│  │ 2. sequence load                ~3-10 cycles                │    │
│  │ 3. CAS (enqueue_pos)            ~15-50 cycles               │    │
│  │    - 성공 시: 계속                                           │    │
│  │    - 실패 시: 1번으로 돌아감 (+ 캐시 무효화 비용)             │    │
│  │ 4. data store                   ~1 cycle                    │    │
│  │ 5. sequence store (release)     ~1 cycle                    │    │
│  │ ─────────────────────────────────                           │    │
│  │ 총 (성공 시): ~20-65 cycles (~7-20 ns)                       │    │
│  │ 총 (1회 재시도): ~50-150 cycles (~15-50 ns)                  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### NUMA 환경에서의 Queue 최적화

```cpp
// NUMA 인지 Queue 할당
#include <numa.h>

MPMCQueue* create_numa_queue(size_t capacity, int node) {
    // 특정 NUMA 노드에 메모리 할당
    void* mem = numa_alloc_onnode(sizeof(MPMCQueue), node);
    MPMCQueue* q = (MPMCQueue*)mem;

    // buffer도 같은 노드에 할당
    q->buffer = numa_alloc_onnode(capacity * sizeof(Cell), node);

    // ...
    return q;
}

// Producer/Consumer를 Queue와 같은 NUMA 노드에 바인딩
void bind_to_node(int node) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);

    // 해당 노드의 CPU들만 포함
    struct bitmask* cpus = numa_allocate_cpumask();
    numa_node_to_cpus(node, cpus);

    for (int i = 0; i < numa_num_configured_cpus(); i++) {
        if (numa_bitmask_isbitset(cpus, i)) {
            CPU_SET(i, &cpuset);
        }
    }

    pthread_setaffinity_np(pthread_self(), sizeof(cpuset), &cpuset);
    numa_free_cpumask(cpus);
}
```

---

## 관련 문서

- [CAS 연산](./01-cas-operation.md) - Queue 구현의 기반
- [Memory Ordering](./02-memory-ordering.md) - acquire/release 패턴
- [ABA Problem](./07-aba-problem.md) - Unbounded Queue 주의점
- [Lock-Free Queue](./04-lock-free-queue.md) - Michael-Scott Queue 상세

---

## 참고 자료

- [1024cores - Lock-Free MPSC Queue](http://www.1024cores.net/home/lock-free-algorithms/queues/non-intrusive-mpsc-node-based-queue)
- [Bounded MPMC Queue - Dmitry Vyukov](https://www.1024cores.net/home/lock-free-algorithms/queues/bounded-mpmc-queue)
- [Folly ProducerConsumerQueue](https://github.com/facebook/folly/blob/main/folly/ProducerConsumerQueue.h)
- [moodycamel::ConcurrentQueue](https://github.com/cameron314/concurrentqueue)
