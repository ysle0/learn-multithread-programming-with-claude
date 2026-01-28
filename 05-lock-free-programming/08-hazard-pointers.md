# Hazard Pointers

## 목차
1. [개요](#개요)
2. [동작 원리](#동작-원리)
3. [기본 구현](#기본-구현)
4. [최적화 기법](#최적화-기법)
5. [실전 예제](#실전-예제)
6. [대안 기법 비교](#대안-기법-비교)
7. [요약](#요약)

---

## 개요

**Hazard Pointers**는 Lock-Free 자료구조에서 안전한 메모리 회수(Safe Memory Reclamation)를 위한 기법입니다. Maged M. Michael이 2004년에 제안했으며, ABA 문제와 Use-After-Free를 방지합니다.

### 핵심 아이디어

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hazard Pointers 핵심 개념                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  문제: 언제 노드를 안전하게 해제할 수 있는가?                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Thread 1: pop() → 노드 A를 가져옴                        │   │
│  │ Thread 2: 노드 A를 해제하려 함                           │   │
│  │                                                          │   │
│  │ T2가 A를 해제하면 T1은 Use-After-Free!                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  해결: "내가 이 포인터 쓰고 있어!" 공개적으로 선언               │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Thread 1: HP[0] = A  (Hazard Pointer에 A 등록)           │   │
│  │ Thread 2: A 해제 전 모든 HP 확인                         │   │
│  │           → HP에 A가 있음 → 해제 보류                    │   │
│  │ Thread 1: HP[0] = NULL (A 사용 끝)                       │   │
│  │ Thread 2: 다시 확인 → HP에 A 없음 → 안전하게 해제        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 용어 정의

| 용어 | 설명 |
|------|------|
| **Hazard Pointer (HP)** | 스레드가 현재 접근 중인 포인터를 저장하는 공유 변수 |
| **Protected** | HP에 등록된 포인터 (해제 불가) |
| **Retired** | 논리적으로 삭제되었지만 아직 해제되지 않은 노드 |
| **Reclaim** | retired 노드를 실제로 해제하는 과정 |

---

## 동작 원리

### 전체 프로세스

```
┌─────────────────────────────────────────────────────────────────┐
│                 Hazard Pointers 동작 흐름                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Reader 스레드]                                                 │
│                                                                 │
│  1. 포인터 읽기 전:                                              │
│     ┌─────────────────────────────────────────────┐            │
│     │ ptr = load(target)                          │            │
│     │ HP[slot] = ptr        // HP에 등록          │            │
│     │ if (load(target) != ptr)  // 재확인         │            │
│     │     goto retry                              │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  2. 포인터 사용:                                                 │
│     ┌─────────────────────────────────────────────┐            │
│     │ // ptr을 안전하게 역참조 가능                 │            │
│     │ data = ptr->data                            │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  3. 사용 완료 후:                                                │
│     ┌─────────────────────────────────────────────┐            │
│     │ HP[slot] = NULL       // HP 해제            │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  [Writer 스레드 - 노드 삭제 시]                                  │
│                                                                 │
│  1. 노드를 자료구조에서 제거:                                    │
│     ┌─────────────────────────────────────────────┐            │
│     │ CAS로 노드 unlink                            │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  2. retired 리스트에 추가:                                       │
│     ┌─────────────────────────────────────────────┐            │
│     │ retired_list.add(node)                      │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
│  3. 주기적으로 scan & reclaim:                                   │
│     ┌─────────────────────────────────────────────┐            │
│     │ for each node in retired_list:              │            │
│     │     if (node not in any HP)                 │            │
│     │         free(node)                          │            │
│     │     else                                    │            │
│     │         keep in retired_list                │            │
│     └─────────────────────────────────────────────┘            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 메모리 배리어 요구사항

```
┌─────────────────────────────────────────────────────────────────┐
│                Memory Ordering 요구사항                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HP 설정 (Reader):                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ptr = load(target, acquire);                             │   │
│  │ store(HP[slot], ptr, release);  // HP 설정              │   │
│  │ fence(seq_cst);                 // 필수!                │   │
│  │ if (load(target, acquire) != ptr) retry;                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  왜 fence가 필요한가?                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ HP 설정이 재확인 load보다 먼저 다른 스레드에 보여야 함   │   │
│  │ 그래야 Writer가 HP를 확인할 때 최신 값을 봄              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  HP 스캔 (Writer):                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ fence(seq_cst);                 // 필수!                │   │
│  │ for each thread t:                                       │   │
│  │     hp = load(HP[t][slot], acquire);                    │   │
│  │     if (hp == node) return PROTECTED;                    │   │
│  │ return SAFE_TO_FREE;                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 기본 구현

### 자료구조 정의

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdbool.h>
#include <string.h>

#define MAX_THREADS 64
#define HP_PER_THREAD 2
#define RETIRE_THRESHOLD 100

typedef struct Node {
    void* data;
    struct Node* next;
} Node;

// Hazard Pointer 레코드
typedef struct {
    _Atomic(Node*) hp[HP_PER_THREAD];
} HPRecord;

// 전역 HP 배열
HPRecord hp_records[MAX_THREADS];

// Retired 노드 리스트
typedef struct RetiredNode {
    Node* node;
    struct RetiredNode* next;
} RetiredNode;

// 스레드 로컬 데이터
typedef struct {
    int thread_id;
    RetiredNode* retired_list;
    int retired_count;
} ThreadData;

__thread ThreadData* tls_data = NULL;
```

### HP 관리 함수

```c
// 스레드 초기화
void hp_thread_init(int tid) {
    tls_data = (ThreadData*)malloc(sizeof(ThreadData));
    tls_data->thread_id = tid;
    tls_data->retired_list = NULL;
    tls_data->retired_count = 0;

    // HP 초기화
    for (int i = 0; i < HP_PER_THREAD; i++) {
        atomic_store(&hp_records[tid].hp[i], NULL);
    }
}

// HP 설정 (acquire 시맨틱)
Node* hp_protect(int slot, _Atomic(Node*)* target) {
    Node* ptr;
    Node* hp;

    do {
        ptr = atomic_load_explicit(target, memory_order_acquire);
        atomic_store_explicit(&hp_records[tls_data->thread_id].hp[slot],
                              ptr, memory_order_release);

        // 메모리 배리어 (HP 설정이 다른 스레드에 보이도록)
        atomic_thread_fence(memory_order_seq_cst);

        // 재확인: target이 바뀌지 않았는지
        hp = atomic_load_explicit(target, memory_order_acquire);
    } while (ptr != hp);

    return ptr;
}

// HP 해제
void hp_clear(int slot) {
    atomic_store_explicit(&hp_records[tls_data->thread_id].hp[slot],
                          NULL, memory_order_release);
}

// 노드가 보호되고 있는지 확인
bool is_protected(Node* node) {
    atomic_thread_fence(memory_order_seq_cst);

    for (int t = 0; t < MAX_THREADS; t++) {
        for (int h = 0; h < HP_PER_THREAD; h++) {
            Node* hp = atomic_load_explicit(&hp_records[t].hp[h],
                                            memory_order_acquire);
            if (hp == node) {
                return true;
            }
        }
    }
    return false;
}
```

### Retire와 Reclaim

```c
// 노드를 retired 리스트에 추가
void hp_retire(Node* node) {
    RetiredNode* rn = (RetiredNode*)malloc(sizeof(RetiredNode));
    rn->node = node;
    rn->next = tls_data->retired_list;
    tls_data->retired_list = rn;
    tls_data->retired_count++;

    // 임계치 도달 시 reclaim 시도
    if (tls_data->retired_count >= RETIRE_THRESHOLD) {
        hp_scan();
    }
}

// 보호되지 않은 노드 해제
void hp_scan() {
    // 1. 모든 HP 수집
    Node* protected_nodes[MAX_THREADS * HP_PER_THREAD];
    int pcount = 0;

    for (int t = 0; t < MAX_THREADS; t++) {
        for (int h = 0; h < HP_PER_THREAD; h++) {
            Node* hp = atomic_load_explicit(&hp_records[t].hp[h],
                                            memory_order_acquire);
            if (hp != NULL) {
                protected_nodes[pcount++] = hp;
            }
        }
    }

    // 2. retired 리스트 순회하며 해제 가능한 노드 찾기
    RetiredNode** curr = &tls_data->retired_list;

    while (*curr != NULL) {
        Node* node = (*curr)->node;
        bool safe_to_free = true;

        // 보호된 노드인지 확인
        for (int i = 0; i < pcount; i++) {
            if (protected_nodes[i] == node) {
                safe_to_free = false;
                break;
            }
        }

        if (safe_to_free) {
            // 안전하게 해제
            RetiredNode* to_free = *curr;
            *curr = (*curr)->next;
            free(to_free->node);
            free(to_free);
            tls_data->retired_count--;
        } else {
            // 다음 노드로
            curr = &(*curr)->next;
        }
    }
}

// 스레드 종료 시 정리
void hp_thread_cleanup() {
    // 남은 retired 노드 처리
    while (tls_data->retired_list != NULL) {
        hp_scan();
        // 아직 보호 중인 노드가 있으면 대기
        // 실제 구현에서는 다른 스레드에 넘기거나 글로벌 리스트 사용
    }

    // HP 클리어
    for (int i = 0; i < HP_PER_THREAD; i++) {
        hp_clear(i);
    }

    free(tls_data);
    tls_data = NULL;
}
```

### HP 기반 Lock-Free Stack

```c
typedef struct {
    _Atomic(Node*) top;
} HPStack;

void hp_stack_init(HPStack* stack) {
    atomic_store(&stack->top, NULL);
}

void hp_stack_push(HPStack* stack, void* data) {
    Node* new_node = (Node*)malloc(sizeof(Node));
    new_node->data = data;

    Node* old_top;
    do {
        old_top = atomic_load(&stack->top);
        new_node->next = old_top;
    } while (!atomic_compare_exchange_weak(&stack->top, &old_top, new_node));
}

void* hp_stack_pop(HPStack* stack) {
    Node* old_top;
    Node* new_top;

    while (true) {
        // HP로 top 보호
        old_top = hp_protect(0, &stack->top);

        if (old_top == NULL) {
            hp_clear(0);
            return NULL;
        }

        // next도 보호 (optional, 더 안전)
        new_top = hp_protect(1, &old_top->next);

        // CAS로 pop 시도
        if (atomic_compare_exchange_weak(&stack->top, &old_top, new_top)) {
            void* data = old_top->data;

            // HP 해제 후 retire
            hp_clear(0);
            hp_clear(1);
            hp_retire(old_top);

            return data;
        }

        // 실패 시 HP 해제하고 재시도
        hp_clear(0);
        hp_clear(1);
    }
}
```

---

## 최적화 기법

### 1. HP 슬롯 캐싱

```c
// HP 슬롯을 미리 할당해두고 재사용
typedef struct {
    int slot;
    bool in_use;
} HPSlot;

__thread HPSlot slot_cache[HP_PER_THREAD] = {{0, false}, {1, false}};

int acquire_hp_slot() {
    for (int i = 0; i < HP_PER_THREAD; i++) {
        if (!slot_cache[i].in_use) {
            slot_cache[i].in_use = true;
            return slot_cache[i].slot;
        }
    }
    return -1;  // 슬롯 부족
}

void release_hp_slot(int slot) {
    for (int i = 0; i < HP_PER_THREAD; i++) {
        if (slot_cache[i].slot == slot) {
            hp_clear(slot);
            slot_cache[i].in_use = false;
            return;
        }
    }
}
```

### 2. Batch Reclamation

```c
#define BATCH_SIZE 32

void hp_scan_batch() {
    // 모든 HP를 한 번에 수집 (캐시 효율)
    Node* protected_set[MAX_THREADS * HP_PER_THREAD];
    int pcount = 0;

    // HP 수집
    for (int t = 0; t < MAX_THREADS; t++) {
        for (int h = 0; h < HP_PER_THREAD; h++) {
            Node* hp = atomic_load(&hp_records[t].hp[h]);
            if (hp != NULL) {
                protected_set[pcount++] = hp;
            }
        }
    }

    // 정렬하여 이진 탐색 가능하게 (많은 retired 노드가 있을 때 효율적)
    qsort(protected_set, pcount, sizeof(Node*), ptr_compare);

    // Batch 단위로 처리
    RetiredNode* batch[BATCH_SIZE];
    int batch_count = 0;

    RetiredNode** curr = &tls_data->retired_list;
    while (*curr != NULL) {
        batch[batch_count++] = *curr;

        if (batch_count == BATCH_SIZE || (*curr)->next == NULL) {
            // 배치 처리
            for (int i = 0; i < batch_count; i++) {
                bool found = bsearch(&batch[i]->node, protected_set, pcount,
                                     sizeof(Node*), ptr_compare) != NULL;
                if (!found) {
                    // 해제
                    free(batch[i]->node);
                    // 리스트에서 제거 로직...
                }
            }
            batch_count = 0;
        }

        curr = &(*curr)->next;
    }
}
```

### 3. 스레드별 HP 레코드 지역화

```c
// 캐시 라인 정렬로 false sharing 방지
typedef struct {
    _Atomic(Node*) hp[HP_PER_THREAD];
    char padding[64 - (HP_PER_THREAD * sizeof(Node*)) % 64];
} HPRecordAligned __attribute__((aligned(64)));

HPRecordAligned hp_records[MAX_THREADS];
```

### 4. 동적 스레드 관리

```c
// 스레드 등록/해제 지원
typedef struct {
    _Atomic(bool) active;
    _Atomic(Node*) hp[HP_PER_THREAD];
} DynamicHPRecord;

DynamicHPRecord hp_records[MAX_THREADS];
_Atomic(int) thread_count = 0;

int hp_register_thread() {
    for (int i = 0; i < MAX_THREADS; i++) {
        bool expected = false;
        if (atomic_compare_exchange_strong(&hp_records[i].active,
                                           &expected, true)) {
            // HP 초기화
            for (int h = 0; h < HP_PER_THREAD; h++) {
                atomic_store(&hp_records[i].hp[h], NULL);
            }
            atomic_fetch_add(&thread_count, 1);
            return i;
        }
    }
    return -1;  // 슬롯 없음
}

void hp_unregister_thread(int tid) {
    // HP 클리어
    for (int h = 0; h < HP_PER_THREAD; h++) {
        atomic_store(&hp_records[tid].hp[h], NULL);
    }
    atomic_store(&hp_records[tid].active, false);
    atomic_fetch_sub(&thread_count, 1);
}

// 스캔 시 활성 스레드만 확인
bool is_protected_dynamic(Node* node) {
    for (int t = 0; t < MAX_THREADS; t++) {
        if (!atomic_load(&hp_records[t].active)) continue;

        for (int h = 0; h < HP_PER_THREAD; h++) {
            if (atomic_load(&hp_records[t].hp[h]) == node) {
                return true;
            }
        }
    }
    return false;
}
```

---

## 실전 예제

### HP 기반 Lock-Free Queue (Michael-Scott Queue)

```c
typedef struct QNode {
    void* data;
    _Atomic(struct QNode*) next;
} QNode;

typedef struct {
    _Atomic(QNode*) head;
    _Atomic(QNode*) tail;
} HPQueue;

void hp_queue_init(HPQueue* queue) {
    QNode* dummy = (QNode*)malloc(sizeof(QNode));
    dummy->data = NULL;
    atomic_store(&dummy->next, NULL);
    atomic_store(&queue->head, dummy);
    atomic_store(&queue->tail, dummy);
}

void hp_queue_enqueue(HPQueue* queue, void* data) {
    QNode* new_node = (QNode*)malloc(sizeof(QNode));
    new_node->data = data;
    atomic_store(&new_node->next, NULL);

    while (true) {
        // tail 보호
        QNode* tail = hp_protect(0, &queue->tail);
        QNode* next = atomic_load(&tail->next);

        if (tail == atomic_load(&queue->tail)) {
            if (next == NULL) {
                // tail의 next에 새 노드 연결 시도
                if (atomic_compare_exchange_weak(&tail->next, &next, new_node)) {
                    // tail 업데이트 시도 (실패해도 OK)
                    atomic_compare_exchange_weak(&queue->tail, &tail, new_node);
                    hp_clear(0);
                    return;
                }
            } else {
                // tail이 뒤쳐져 있음, 따라잡기
                atomic_compare_exchange_weak(&queue->tail, &tail, next);
            }
        }
    }
}

void* hp_queue_dequeue(HPQueue* queue) {
    while (true) {
        // head와 tail 보호
        QNode* head = hp_protect(0, &queue->head);
        QNode* tail = atomic_load(&queue->tail);
        QNode* next = hp_protect(1, &head->next);

        if (head == atomic_load(&queue->head)) {
            if (head == tail) {
                if (next == NULL) {
                    // 큐가 비어있음
                    hp_clear(0);
                    hp_clear(1);
                    return NULL;
                }
                // tail이 뒤쳐져 있음
                atomic_compare_exchange_weak(&queue->tail, &tail, next);
            } else {
                // 데이터 먼저 읽기 (head 삭제 전)
                void* data = next->data;

                if (atomic_compare_exchange_weak(&queue->head, &head, next)) {
                    hp_clear(0);
                    hp_clear(1);
                    hp_retire(head);  // dummy 노드 retire
                    return data;
                }
            }
        }

        hp_clear(0);
        hp_clear(1);
    }
}
```

### HP 기반 Lock-Free Hash Map (Bucket List)

```c
typedef struct HashNode {
    uint64_t key;
    void* value;
    _Atomic(struct HashNode*) next;
} HashNode;

typedef struct {
    _Atomic(HashNode*) buckets[BUCKET_COUNT];
} HPHashMap;

void* hp_hashmap_find(HPHashMap* map, uint64_t key) {
    int bucket = hash(key) % BUCKET_COUNT;

    HashNode* curr = hp_protect(0, &map->buckets[bucket]);

    while (curr != NULL) {
        if (curr->key == key) {
            void* value = curr->value;
            hp_clear(0);
            return value;
        }

        // 다음 노드로 이동 (HP 전환)
        HashNode* next = hp_protect(1, &curr->next);
        hp_clear(0);

        // HP 슬롯 교체
        atomic_store(&hp_records[tls_data->thread_id].hp[0],
                     atomic_load(&hp_records[tls_data->thread_id].hp[1]));
        hp_clear(1);

        curr = next;
    }

    hp_clear(0);
    return NULL;
}

bool hp_hashmap_insert(HPHashMap* map, uint64_t key, void* value) {
    int bucket = hash(key) % BUCKET_COUNT;

    HashNode* new_node = (HashNode*)malloc(sizeof(HashNode));
    new_node->key = key;
    new_node->value = value;

    while (true) {
        HashNode* head = atomic_load(&map->buckets[bucket]);
        atomic_store(&new_node->next, head);

        if (atomic_compare_exchange_weak(&map->buckets[bucket], &head, new_node)) {
            return true;
        }
    }
}

bool hp_hashmap_remove(HPHashMap* map, uint64_t key) {
    int bucket = hash(key) % BUCKET_COUNT;

    while (true) {
        _Atomic(HashNode*)* prev = &map->buckets[bucket];
        HashNode* curr = hp_protect(0, prev);

        while (curr != NULL) {
            if (curr->key == key) {
                HashNode* next = atomic_load(&curr->next);

                if (atomic_compare_exchange_weak(prev, &curr, next)) {
                    hp_clear(0);
                    hp_retire(curr);
                    return true;
                }
                break;  // CAS 실패, 재시도
            }

            // 다음 노드로
            prev = &curr->next;
            HashNode* next = hp_protect(1, prev);
            hp_clear(0);

            atomic_store(&hp_records[tls_data->thread_id].hp[0],
                         atomic_load(&hp_records[tls_data->thread_id].hp[1]));
            hp_clear(1);

            curr = next;
        }

        if (curr == NULL) {
            hp_clear(0);
            return false;  // 키 없음
        }
    }
}
```

---

## 대안 기법 비교

### Hazard Pointers vs Epoch-Based Reclamation

```
┌─────────────────────────────────────────────────────────────────┐
│              HP vs EBR 비교                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Hazard Pointers:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 장점:                                                     │   │
│  │ - 개별 포인터 단위 보호 (정밀)                            │   │
│  │ - Worst-case 메모리 bounded (O(T*H + T*R))              │   │
│  │ - 긴 작업이 다른 스레드에 영향 주지 않음                  │   │
│  │                                                          │   │
│  │ 단점:                                                     │   │
│  │ - 모든 HP 스캔 필요 (O(T*H) per reclaim)                 │   │
│  │ - 메모리 배리어 많음                                      │   │
│  │ - 구현 복잡                                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Epoch-Based Reclamation:                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 장점:                                                     │   │
│  │ - 구현 간단                                               │   │
│  │ - 오버헤드 낮음 (epoch 증가만)                            │   │
│  │ - 빠른 경로에 배리어 적음                                 │   │
│  │                                                          │   │
│  │ 단점:                                                     │   │
│  │ - 한 스레드가 멈추면 전체 메모리 해제 지연                 │   │
│  │ - Worst-case 메모리 unbounded                            │   │
│  │ - 긴 작업에 부적합                                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  선택 가이드:                                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 짧은 임계 구역, 높은 처리량 → EBR                       │   │
│  │ - 긴 작업, 메모리 제약 → HP                              │   │
│  │ - Rust (crossbeam) → EBR                                 │   │
│  │ - 범용 라이브러리 → HP                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Reference Counting vs HP

```
┌─────────────────────────────────────────────────────────────────┐
│              Reference Counting vs HP                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Reference Counting:                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 장점: 직관적, 표준 라이브러리 지원 (shared_ptr)          │   │
│  │ 단점: 매 접근마다 카운터 수정 → 캐시 라인 경합            │   │
│  │       순환 참조 문제                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Hazard Pointers:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 장점: 읽기 경로에 쓰기 없음 (HP 설정 제외)                │   │
│  │       순환 참조 문제 없음                                 │   │
│  │ 단점: 구현 복잡, scan 오버헤드                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  성능 비교 (읽기 위주 워크로드):                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ HP > EBR > Reference Counting                            │   │
│  │ (HP가 읽기 시 가장 적은 원자적 연산)                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 요약

### HP 핵심 정리

| 항목 | 설명 |
|------|------|
| **목적** | Lock-Free에서 안전한 메모리 회수 |
| **원리** | 접근 중인 포인터를 공개하여 해제 방지 |
| **장점** | ABA 해결, bounded 메모리, 긴 작업 OK |
| **단점** | scan 오버헤드, 구현 복잡, 메모리 배리어 |

### HP 사용 체크리스트

```
┌─────────────────────────────────────────────────────────────────┐
│                HP 사용 체크리스트                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  □ 포인터 읽기 전 HP 설정                                       │
│  □ HP 설정 후 재확인 (target이 안 바뀌었는지)                    │
│  □ 포인터 사용 완료 후 HP 해제                                   │
│  □ 노드 삭제 시 retire (바로 free 금지!)                         │
│  □ 주기적으로 scan & reclaim                                    │
│  □ 스레드 종료 시 정리                                          │
│  □ 적절한 메모리 배리어 사용                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 복잡도

| 연산 | 시간 복잡도 |
|------|------------|
| HP 설정 | O(1) |
| HP 해제 | O(1) |
| Retire | O(1) |
| Scan | O(T*H + R) |

T = 스레드 수, H = HP per thread, R = retired 노드 수

---

## 관련 문서

- [ABA Problem](./07-aba-problem.md) - HP가 해결하는 문제
- [CAS 연산](./01-cas-operation.md) - HP와 함께 사용되는 연산
- [Lock-Free Stack](./03-lock-free-stack.md) - HP 적용 예
- [Lock-Free Queue](./04-lock-free-queue.md) - HP 적용 예

---

## 참고 자료

- [Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects - Maged M. Michael](https://www.research.ibm.com/people/m/michael/ieeetpds-2004.pdf)
- [Lock-Free Data Structures with Hazard Pointers - Dr. Dobb's](https://www.drdobbs.com/lock-free-data-structures-with-hazard-lk/184401890)
- [Folly Hazard Pointers Implementation](https://github.com/facebook/folly/blob/main/folly/synchronization/HazardPointer.h)
- [libcds Hazard Pointers](http://libcds.sourceforge.net/doc/cds-api/group__cds__gc__hp.html)
