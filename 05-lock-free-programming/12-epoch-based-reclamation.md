# Epoch-based Reclamation (EBR)

## 1. EBR 개요

### 1.1 Epoch-based Reclamation이란?

Epoch-based Reclamation(EBR)은 **락-프리 데이터 구조에서 안전하게 메모리를 해제**하기 위한 기법입니다. Hazard Pointer의 대안으로, 특히 **Rust의 Crossbeam 라이브러리**에서 사용됩니다.

```
EBR 핵심 아이디어:

시간을 "Epoch"라는 논리적 단위로 나눔

Epoch 0          Epoch 1          Epoch 2
├──────────────┼──────────────┼──────────────┤
│ Objects A, B │ Objects C, D │ Objects E, F │
│   retired    │   retired    │   retired    │
└──────────────┴──────────────┴──────────────┘

규칙:
- Epoch N에서 retire된 객체는 Epoch N+2에서 해제 가능
- 모든 스레드가 Epoch N 이상이면 Epoch N-2 객체 해제 가능
```

### 1.2 Hazard Pointer vs EBR

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│       특성          │   Hazard Pointer    │        EBR          │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 보호 단위           │   개별 포인터       │   전체 Critical     │
│                     │                     │   Section           │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 메모리 사용         │   포인터당 HP 필요  │   스레드당 epoch    │
│                     │                     │   카운터만          │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 오버헤드            │   HP 설정/해제      │   epoch 진입/퇴장   │
│                     │   (높음)            │   (낮음)            │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 해제 지연           │   즉시 (retire 후)  │   2 epoch 대기      │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 재진입 위험         │   없음              │   주의 필요         │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ 구현 복잡도         │   높음              │   중간              │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

## 2. EBR 동작 원리

### 2.1 Global Epoch

```c
// 전역 epoch 카운터
atomic_uint global_epoch = 0;

// Epoch 전이 조건
// 모든 활성 스레드가 현재 global_epoch를 본 경우에만 진행

Thread 0: [----Epoch 1----][----Epoch 2----][----Epoch 3----]
Thread 1:     [----Epoch 1----][----Epoch 2----][----Epoch 3----]
Thread 2:         [----Epoch 1----][----Epoch 2----]
Global:    1 → → → → → → → → 2 → → → → → → → → 3 → → → → → →

Epoch 2로 전이하려면: 모든 스레드가 Epoch 1 이상
Epoch 3로 전이하려면: 모든 스레드가 Epoch 2 이상
```

### 2.2 Thread-Local State

```c
// 각 스레드의 상태
struct thread_state {
    atomic_uint local_epoch;  // 현재 스레드가 본 epoch
    bool is_active;           // Critical Section 내부인지
    garbage_list bags[3];     // Epoch별 retire 목록
};

// 3개의 가비지 목록을 사용하는 이유:
// - bags[0]: 현재 epoch에서 retire
// - bags[1]: 이전 epoch에서 retire
// - bags[2]: 2 epoch 전에 retire (해제 가능!)
```

### 2.3 기본 동작 흐름

```
1. Critical Section 진입
   - local_epoch = global_epoch 로드
   - is_active = true

2. 데이터 접근
   - 안전하게 포인터 역참조

3. 객체 Retire (삭제 요청)
   - bags[global_epoch % 3]에 추가

4. Critical Section 퇴장
   - is_active = false
   - 가능하면 메모리 해제 시도

5. Epoch 전이 시도
   - 모든 스레드 확인
   - 조건 충족 시 global_epoch++
   - 오래된 bag 해제
```

## 3. EBR 구현

### 3.1 기본 자료 구조

```c
#include <stdatomic.h>
#include <stdbool.h>
#include <stdlib.h>

#define MAX_THREADS 64
#define NUM_EPOCHS 3

// 가비지 리스트 노드
typedef struct garbage_node {
    void *ptr;
    void (*destructor)(void *);
    struct garbage_node *next;
} garbage_node_t;

// 가비지 백 (Epoch별 retire 목록)
typedef struct {
    garbage_node_t *head;
    size_t count;
} garbage_bag_t;

// 스레드별 상태
typedef struct {
    _Alignas(64) atomic_uint local_epoch;
    _Alignas(64) atomic_bool is_active;
    garbage_bag_t bags[NUM_EPOCHS];
} thread_state_t;

// 전역 상태
static atomic_uint global_epoch = 0;
static thread_state_t threads[MAX_THREADS];
static __thread int thread_id = -1;
```

### 3.2 스레드 등록/해제

```c
static atomic_int next_thread_id = 0;

void ebr_register_thread(void)
{
    thread_id = atomic_fetch_add(&next_thread_id, 1);

    if (thread_id >= MAX_THREADS) {
        fprintf(stderr, "Too many threads!\n");
        abort();
    }

    thread_state_t *state = &threads[thread_id];
    atomic_store(&state->local_epoch, 0);
    atomic_store(&state->is_active, false);

    for (int i = 0; i < NUM_EPOCHS; i++) {
        state->bags[i].head = NULL;
        state->bags[i].count = 0;
    }
}

void ebr_unregister_thread(void)
{
    thread_state_t *state = &threads[thread_id];

    // 모든 가비지 해제
    for (int i = 0; i < NUM_EPOCHS; i++) {
        garbage_node_t *node = state->bags[i].head;
        while (node) {
            garbage_node_t *next = node->next;
            if (node->destructor) {
                node->destructor(node->ptr);
            } else {
                free(node->ptr);
            }
            free(node);
            node = next;
        }
    }

    atomic_store(&state->is_active, false);
    thread_id = -1;
}
```

### 3.3 Critical Section

```c
// Guard - RAII 스타일 보호
typedef struct {
    int tid;
} ebr_guard_t;

// Critical Section 진입
ebr_guard_t ebr_pin(void)
{
    thread_state_t *state = &threads[thread_id];

    // 현재 global epoch 로드
    unsigned epoch = atomic_load_explicit(&global_epoch, memory_order_acquire);
    atomic_store_explicit(&state->local_epoch, epoch, memory_order_relaxed);

    // 활성화
    atomic_store_explicit(&state->is_active, true, memory_order_release);

    // 메모리 펜스 - is_active가 먼저 보이도록
    atomic_thread_fence(memory_order_seq_cst);

    return (ebr_guard_t){ .tid = thread_id };
}

// Critical Section 퇴장
void ebr_unpin(ebr_guard_t *guard)
{
    thread_state_t *state = &threads[guard->tid];

    // 비활성화
    atomic_store_explicit(&state->is_active, false, memory_order_release);

    // 주기적으로 가비지 컬렉션 시도
    ebr_try_collect();
}
```

### 3.4 Retire (삭제 요청)

```c
void ebr_retire(void *ptr, void (*destructor)(void *))
{
    thread_state_t *state = &threads[thread_id];
    unsigned epoch = atomic_load(&global_epoch);

    // 가비지 노드 생성
    garbage_node_t *node = malloc(sizeof(garbage_node_t));
    node->ptr = ptr;
    node->destructor = destructor;

    // 현재 epoch의 bag에 추가
    int bag_idx = epoch % NUM_EPOCHS;
    node->next = state->bags[bag_idx].head;
    state->bags[bag_idx].head = node;
    state->bags[bag_idx].count++;
}

// 기본 free() 사용
void ebr_retire_free(void *ptr)
{
    ebr_retire(ptr, NULL);
}
```

### 3.5 Epoch 전이 및 가비지 컬렉션

```c
// 모든 스레드가 특정 epoch 이상인지 확인
static bool all_threads_past_epoch(unsigned target_epoch)
{
    int num_threads = atomic_load(&next_thread_id);

    for (int i = 0; i < num_threads; i++) {
        thread_state_t *state = &threads[i];

        // 비활성 스레드는 건너뜀
        if (!atomic_load_explicit(&state->is_active, memory_order_acquire)) {
            continue;
        }

        unsigned local = atomic_load_explicit(&state->local_epoch,
                                               memory_order_acquire);
        if (local < target_epoch) {
            return false;
        }
    }

    return true;
}

// Epoch 전이 시도
static bool try_advance_epoch(void)
{
    unsigned current = atomic_load(&global_epoch);

    // 모든 활성 스레드가 현재 epoch를 본 경우에만 진행
    if (!all_threads_past_epoch(current)) {
        return false;
    }

    // Epoch 증가 (CAS로 한 스레드만 성공)
    return atomic_compare_exchange_strong(&global_epoch, &current, current + 1);
}

// 가비지 컬렉션
void ebr_try_collect(void)
{
    // Epoch 전이 시도
    if (try_advance_epoch()) {
        // 전이 성공 - 오래된 가비지 해제
        unsigned current = atomic_load(&global_epoch);

        // 2 epoch 전의 bag은 안전하게 해제 가능
        int old_bag_idx = (current + 1) % NUM_EPOCHS;  // 2 epoch 전

        // 모든 스레드의 오래된 bag 해제
        int num_threads = atomic_load(&next_thread_id);
        for (int i = 0; i < num_threads; i++) {
            garbage_bag_t *bag = &threads[i].bags[old_bag_idx];

            garbage_node_t *node = bag->head;
            while (node) {
                garbage_node_t *next = node->next;
                if (node->destructor) {
                    node->destructor(node->ptr);
                } else {
                    free(node->ptr);
                }
                free(node);
                node = next;
            }

            bag->head = NULL;
            bag->count = 0;
        }
    }
}
```

## 4. Crossbeam EBR

### 4.1 Crossbeam-epoch 개요

Rust의 Crossbeam 라이브러리는 가장 널리 사용되는 EBR 구현입니다.

```rust
use crossbeam_epoch::{self as epoch, Atomic, Owned, Shared};
use std::sync::atomic::Ordering;

// Atomic 포인터 타입
struct Node<T> {
    data: T,
    next: Atomic<Node<T>>,
}

// 스택 구현 예시
struct Stack<T> {
    head: Atomic<Node<T>>,
}

impl<T> Stack<T> {
    fn new() -> Self {
        Stack {
            head: Atomic::null(),
        }
    }

    fn push(&self, data: T) {
        let node = Owned::new(Node {
            data,
            next: Atomic::null(),
        });

        // Epoch 보호 획득
        let guard = epoch::pin();

        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            node.next.store(head, Ordering::Relaxed);

            match self.head.compare_exchange(
                head,
                node,
                Ordering::Release,
                Ordering::Relaxed,
                &guard,
            ) {
                Ok(_) => break,
                Err(e) => node = e.new,
            }
        }
    }

    fn pop(&self) -> Option<T> {
        let guard = epoch::pin();

        loop {
            let head = self.head.load(Ordering::Acquire, &guard);

            match unsafe { head.as_ref() } {
                None => return None,
                Some(h) => {
                    let next = h.next.load(Ordering::Relaxed, &guard);

                    if self.head
                        .compare_exchange(
                            head,
                            next,
                            Ordering::Release,
                            Ordering::Relaxed,
                            &guard,
                        )
                        .is_ok()
                    {
                        // 안전하게 retire
                        unsafe {
                            guard.defer_destroy(head);
                        }
                        return Some(ptr::read(&h.data));
                    }
                }
            }
        }
    }
}
```

### 4.2 Crossbeam API 상세

```rust
use crossbeam_epoch::{self as epoch, Atomic, Guard, Owned, Shared};

// 1. Guard - Critical Section
fn guard_example() {
    // 방법 1: pin()
    let guard = epoch::pin();
    // guard가 drop될 때 자동으로 unpin

    // 방법 2: 명시적 unpin
    let guard = epoch::pin();
    drop(guard);  // 또는 스코프 종료
}

// 2. Atomic<T> - 원자적 포인터
fn atomic_example() {
    let atomic: Atomic<i32> = Atomic::null();
    let guard = epoch::pin();

    // 로드
    let ptr: Shared<i32> = atomic.load(Ordering::Acquire, &guard);

    // 저장
    let owned = Owned::new(42);
    atomic.store(owned, Ordering::Release);

    // CAS
    let current = atomic.load(Ordering::Acquire, &guard);
    let new = Owned::new(100);
    let result = atomic.compare_exchange(
        current,
        new,
        Ordering::AcqRel,
        Ordering::Acquire,
        &guard,
    );
}

// 3. Owned<T> - 소유권 있는 포인터
fn owned_example() {
    let owned: Owned<String> = Owned::new(String::from("hello"));

    // Shared로 변환 (guard 필요)
    let guard = epoch::pin();
    let shared: Shared<String> = owned.into_shared(&guard);
}

// 4. Shared<T> - 공유 참조
fn shared_example() {
    let atomic: Atomic<i32> = Atomic::new(42);
    let guard = epoch::pin();

    let shared: Shared<i32> = atomic.load(Ordering::Acquire, &guard);

    // 역참조 (unsafe)
    if !shared.is_null() {
        let value = unsafe { shared.deref() };
        println!("Value: {}", value);
    }

    // 태그 사용
    let tagged = shared.with_tag(1);
    assert_eq!(tagged.tag(), 1);
}

// 5. defer_destroy - 지연 해제
fn defer_example() {
    let atomic: Atomic<String> = Atomic::new(String::from("old"));
    let guard = epoch::pin();

    let old = atomic.swap(
        Owned::new(String::from("new")),
        Ordering::AcqRel,
        &guard,
    );

    // old를 안전하게 retire
    unsafe {
        guard.defer_destroy(old);
    }
    // 2 epoch 후 자동 해제
}
```

### 4.3 Crossbeam 내부 구조

```rust
// Crossbeam 내부 구조 (단순화)

// 전역 상태
struct Global {
    epoch: AtomicUsize,
    // 각 epoch의 가비지 큐
    garbage: [Queue<Deferred>; 3],
}

// 스레드-로컬 상태
struct Local {
    // 현재 스레드의 epoch
    epoch: AtomicUsize,
    // 활성 guard 수
    guard_count: Cell<usize>,
    // 로컬 가비지 백
    bag: UnsafeCell<Bag>,
}

// Guard 구현
struct Guard {
    local: *const Local,
}

impl Guard {
    fn pin() -> Guard {
        let local = LOCAL.with(|l| l.get());

        unsafe {
            let count = (*local).guard_count.get();
            (*local).guard_count.set(count + 1);

            if count == 0 {
                // 첫 pin - epoch 업데이트
                let global_epoch = GLOBAL.epoch.load(Ordering::Relaxed);
                (*local).epoch.store(global_epoch, Ordering::Relaxed);
                atomic::fence(Ordering::SeqCst);
            }
        }

        Guard { local }
    }
}

impl Drop for Guard {
    fn drop(&mut self) {
        unsafe {
            let count = (*self.local).guard_count.get();
            (*self.local).guard_count.set(count - 1);

            if count == 1 {
                // 마지막 unpin - GC 시도
                self.try_advance();
            }
        }
    }
}
```

## 5. 최적화된 EBR 구현

### 5.1 Batch Processing

```c
#define BATCH_SIZE 32

typedef struct {
    void *items[BATCH_SIZE];
    void (*destructors[BATCH_SIZE])(void *);
    size_t count;
} batch_t;

typedef struct {
    _Alignas(64) atomic_uint local_epoch;
    _Alignas(64) atomic_bool is_active;
    batch_t current_batch;
    garbage_bag_t bags[NUM_EPOCHS];
} optimized_thread_state_t;

void ebr_retire_batched(void *ptr, void (*destructor)(void *))
{
    optimized_thread_state_t *state = &threads[thread_id];
    batch_t *batch = &state->current_batch;

    batch->items[batch->count] = ptr;
    batch->destructors[batch->count] = destructor;
    batch->count++;

    // 배치가 가득 차면 bag으로 이동
    if (batch->count >= BATCH_SIZE) {
        flush_batch(state);
    }
}

static void flush_batch(optimized_thread_state_t *state)
{
    unsigned epoch = atomic_load(&global_epoch);
    int bag_idx = epoch % NUM_EPOCHS;
    garbage_bag_t *bag = &state->bags[bag_idx];
    batch_t *batch = &state->current_batch;

    // 배치 내용을 bag으로 이동
    for (size_t i = 0; i < batch->count; i++) {
        garbage_node_t *node = malloc(sizeof(garbage_node_t));
        node->ptr = batch->items[i];
        node->destructor = batch->destructors[i];
        node->next = bag->head;
        bag->head = node;
        bag->count++;
    }

    batch->count = 0;
}
```

### 5.2 Lock-Free Garbage Bag

```c
// Lock-Free 가비지 백
typedef struct garbage_node_lf {
    void *ptr;
    void (*destructor)(void *);
    struct garbage_node_lf *next;
} garbage_node_lf_t;

typedef struct {
    _Alignas(64) atomic_uintptr_t head;
} lockfree_bag_t;

void lockfree_bag_push(lockfree_bag_t *bag, void *ptr, void (*dtor)(void *))
{
    garbage_node_lf_t *node = malloc(sizeof(garbage_node_lf_t));
    node->ptr = ptr;
    node->destructor = dtor;

    garbage_node_lf_t *old_head;
    do {
        old_head = (garbage_node_lf_t *)atomic_load(&bag->head);
        node->next = old_head;
    } while (!atomic_compare_exchange_weak(&bag->head,
                                           (uintptr_t *)&old_head,
                                           (uintptr_t)node));
}

// 전체 리스트를 원자적으로 가져오기
garbage_node_lf_t *lockfree_bag_take_all(lockfree_bag_t *bag)
{
    return (garbage_node_lf_t *)atomic_exchange(&bag->head, 0);
}
```

### 5.3 Adaptive Collection

```c
// 적응형 가비지 컬렉션
typedef struct {
    size_t retired_count;      // 현재 retire된 객체 수
    size_t threshold;          // GC 임계값
    size_t last_collected;     // 마지막 수집량
} gc_stats_t;

static __thread gc_stats_t gc_stats = {
    .threshold = 1000,
};

void ebr_retire_adaptive(void *ptr, void (*destructor)(void *))
{
    ebr_retire(ptr, destructor);
    gc_stats.retired_count++;

    // 임계값 도달 시 GC 시도
    if (gc_stats.retired_count >= gc_stats.threshold) {
        size_t before = gc_stats.retired_count;
        ebr_try_collect();
        size_t collected = before - gc_stats.retired_count;

        // 적응형 임계값 조정
        if (collected < gc_stats.retired_count / 2) {
            // 수집 효율 낮음 - 임계값 증가
            gc_stats.threshold = gc_stats.threshold * 3 / 2;
        } else if (collected > gc_stats.retired_count) {
            // 수집 효율 높음 - 임계값 감소
            gc_stats.threshold = gc_stats.threshold * 2 / 3;
            if (gc_stats.threshold < 100) {
                gc_stats.threshold = 100;
            }
        }

        gc_stats.last_collected = collected;
    }
}
```

## 6. EBR 사용 패턴

### 6.1 Lock-Free 해시맵

```rust
use crossbeam_epoch::{self as epoch, Atomic, Guard, Owned, Shared};
use std::collections::hash_map::DefaultHasher;
use std::hash::{Hash, Hasher};
use std::sync::atomic::Ordering;

struct Bucket<K, V> {
    key: K,
    value: V,
    next: Atomic<Bucket<K, V>>,
}

pub struct HashMap<K, V> {
    buckets: Vec<Atomic<Bucket<K, V>>>,
    size: usize,
}

impl<K: Hash + Eq, V> HashMap<K, V> {
    pub fn new(size: usize) -> Self {
        let mut buckets = Vec::with_capacity(size);
        for _ in 0..size {
            buckets.push(Atomic::null());
        }
        HashMap { buckets, size }
    }

    fn hash(&self, key: &K) -> usize {
        let mut hasher = DefaultHasher::new();
        key.hash(&mut hasher);
        hasher.finish() as usize % self.size
    }

    pub fn get<'g>(&self, key: &K, guard: &'g Guard) -> Option<&'g V> {
        let idx = self.hash(key);
        let mut current = self.buckets[idx].load(Ordering::Acquire, guard);

        while !current.is_null() {
            let node = unsafe { current.deref() };
            if node.key == *key {
                return Some(&node.value);
            }
            current = node.next.load(Ordering::Acquire, guard);
        }

        None
    }

    pub fn insert(&self, key: K, value: V) {
        let guard = epoch::pin();
        let idx = self.hash(&key);

        let new_node = Owned::new(Bucket {
            key,
            value,
            next: Atomic::null(),
        });

        loop {
            let head = self.buckets[idx].load(Ordering::Acquire, &guard);
            new_node.next.store(head, Ordering::Relaxed);

            match self.buckets[idx].compare_exchange(
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

    pub fn remove(&self, key: &K) -> bool {
        let guard = epoch::pin();
        let idx = self.hash(key);

        loop {
            let mut prev = &self.buckets[idx];
            let mut current = prev.load(Ordering::Acquire, &guard);

            while !current.is_null() {
                let node = unsafe { current.deref() };

                if node.key == *key {
                    let next = node.next.load(Ordering::Acquire, &guard);

                    match prev.compare_exchange(
                        current,
                        next,
                        Ordering::Release,
                        Ordering::Relaxed,
                        &guard,
                    ) {
                        Ok(_) => {
                            unsafe { guard.defer_destroy(current); }
                            return true;
                        }
                        Err(_) => break,  // 재시도
                    }
                }

                prev = &node.next;
                current = node.next.load(Ordering::Acquire, &guard);
            }

            // 못 찾음 또는 재시도
            if current.is_null() {
                return false;
            }
        }
    }
}
```

### 6.2 Lock-Free 스킵리스트

```rust
use crossbeam_epoch::{self as epoch, Atomic, Guard, Owned, Shared};
use rand::Rng;

const MAX_LEVEL: usize = 16;

struct Node<K, V> {
    key: K,
    value: V,
    next: [Atomic<Node<K, V>>; MAX_LEVEL],
    level: usize,
}

pub struct SkipList<K, V> {
    head: Atomic<Node<K, V>>,
}

impl<K: Ord, V> SkipList<K, V> {
    pub fn new() -> Self {
        // 센티널 노드
        let head = Owned::new(Node {
            key: unsafe { std::mem::zeroed() },
            value: unsafe { std::mem::zeroed() },
            next: Default::default(),
            level: MAX_LEVEL,
        });

        SkipList {
            head: Atomic::from(head),
        }
    }

    fn random_level() -> usize {
        let mut level = 1;
        let mut rng = rand::thread_rng();
        while level < MAX_LEVEL && rng.gen::<bool>() {
            level += 1;
        }
        level
    }

    pub fn contains(&self, key: &K, guard: &Guard) -> bool {
        let mut current = self.head.load(Ordering::Acquire, guard);

        for level in (0..MAX_LEVEL).rev() {
            loop {
                let node = unsafe { current.deref() };
                let next = node.next[level].load(Ordering::Acquire, guard);

                if next.is_null() {
                    break;
                }

                let next_node = unsafe { next.deref() };
                if next_node.key < *key {
                    current = next;
                } else if next_node.key == *key {
                    return true;
                } else {
                    break;
                }
            }
        }

        false
    }
}
```

### 6.3 Concurrent Queue

```rust
use crossbeam_epoch::{self as epoch, Atomic, Guard, Owned, Shared};
use std::sync::atomic::Ordering;

struct Node<T> {
    data: Option<T>,
    next: Atomic<Node<T>>,
}

pub struct Queue<T> {
    head: Atomic<Node<T>>,
    tail: Atomic<Node<T>>,
}

impl<T> Queue<T> {
    pub fn new() -> Self {
        let sentinel = Owned::new(Node {
            data: None,
            next: Atomic::null(),
        });

        let guard = epoch::pin();
        let sentinel = sentinel.into_shared(&guard);

        Queue {
            head: Atomic::from(sentinel),
            tail: Atomic::from(sentinel),
        }
    }

    pub fn push(&self, data: T) {
        let node = Owned::new(Node {
            data: Some(data),
            next: Atomic::null(),
        });

        let guard = epoch::pin();

        loop {
            let tail = self.tail.load(Ordering::Acquire, &guard);
            let tail_ref = unsafe { tail.deref() };
            let next = tail_ref.next.load(Ordering::Acquire, &guard);

            if next.is_null() {
                match tail_ref.next.compare_exchange(
                    Shared::null(),
                    node,
                    Ordering::Release,
                    Ordering::Relaxed,
                    &guard,
                ) {
                    Ok(node) => {
                        let _ = self.tail.compare_exchange(
                            tail,
                            node,
                            Ordering::Release,
                            Ordering::Relaxed,
                            &guard,
                        );
                        return;
                    }
                    Err(e) => node = e.new,
                }
            } else {
                // tail 뒤처짐 - 업데이트 시도
                let _ = self.tail.compare_exchange(
                    tail,
                    next,
                    Ordering::Release,
                    Ordering::Relaxed,
                    &guard,
                );
            }
        }
    }

    pub fn pop(&self) -> Option<T> {
        let guard = epoch::pin();

        loop {
            let head = self.head.load(Ordering::Acquire, &guard);
            let tail = self.tail.load(Ordering::Acquire, &guard);
            let head_ref = unsafe { head.deref() };
            let next = head_ref.next.load(Ordering::Acquire, &guard);

            if head == tail {
                if next.is_null() {
                    return None;  // 큐 비어있음
                }
                // tail 뒤처짐
                let _ = self.tail.compare_exchange(
                    tail,
                    next,
                    Ordering::Release,
                    Ordering::Relaxed,
                    &guard,
                );
            } else if !next.is_null() {
                let next_ref = unsafe { next.deref() };

                match self.head.compare_exchange(
                    head,
                    next,
                    Ordering::Release,
                    Ordering::Relaxed,
                    &guard,
                ) {
                    Ok(_) => {
                        let data = next_ref.data.take();
                        unsafe { guard.defer_destroy(head); }
                        return data;
                    }
                    Err(_) => continue,
                }
            }
        }
    }
}
```

## 7. 성능 분석

### 7.1 메모리 오버헤드 비교

```
메모리 오버헤드 (객체당):

│  Manual (Free List)
│  ████  8 bytes (next pointer)
│
│  Hazard Pointer
│  ████████████████  24+ bytes (HP records)
│
│  EBR
│  ████████  12 bytes (garbage node)
│
│  Reference Counting
│  ████  8 bytes (counter)
│
└────────────────────────────────────────────
```

### 7.2 연산 오버헤드 비교

```
Critical Section 진입 비용 (나노초):

│  EBR (Crossbeam)
│  ██  ~5-10ns
│
│  Hazard Pointer
│  ████████  ~30-50ns
│
│  RCU
│  ██  ~5-10ns
│
│  Mutex
│  ████████████████████  ~80-100ns
│
└────────────────────────────────────────────
```

### 7.3 확장성 비교

```
처리량 vs 스레드 수:

Throughput (Mops/s)
│
│ 100 ─────────────────────────╱ EBR
│                            ╱
│  80 ─────────────────────╱   HP
│                        ╱   ╱
│  60 ─────────────────╱   ╱
│                    ╱   ╱
│  40 ─────────────╱   ╱
│                ╱   ╱        RWLock
│  20 ─────────╱   ╱    ──────────────
│            ╱   ╱
│   0 ─────╱───╱─────────────────────────→ Threads
          2   4   8   16   32   64
```

### 7.4 메모리 해제 지연

```
retire() 후 실제 해제까지 시간:

│                         │ 최소      │ 평균     │ 최대      │
├─────────────────────────┼───────────┼──────────┼───────────┤
│ Hazard Pointer          │ 즉시      │ ~10μs    │ ~100μs    │
│ EBR                     │ 2 epoch   │ ~50μs    │ ~10ms     │
│ RCU (synchronize)       │ Grace     │ ~50μs    │ ~10ms     │
│ Reference Counting      │ 즉시      │ ~1μs     │ ~10μs     │
```

## 8. EBR 주의사항

### 8.1 일반적인 실수

```rust
// 실수 1: Guard 없이 접근
fn bug1(map: &HashMap<i32, String>) {
    // BUG! guard 없이 load하면 언제든 해제될 수 있음
    let ptr = map.buckets[0].load(Ordering::Acquire, epoch::unprotected());
}

// 실수 2: Guard 범위 밖에서 포인터 사용
fn bug2(map: &HashMap<i32, String>) {
    let value: &String;
    {
        let guard = epoch::pin();
        value = map.get(&42, &guard).unwrap();
    }
    // BUG! guard가 drop되어 value가 무효화됐을 수 있음
    println!("{}", value);
}

// 실수 3: 긴 Critical Section
fn bug3(map: &HashMap<i32, String>) {
    let guard = epoch::pin();

    // 오래 걸리는 작업...
    std::thread::sleep(std::time::Duration::from_secs(10));

    // 이 동안 메모리 해제 불가능 - 메모리 누수!
    drop(guard);
}

// 올바른 방법: 짧은 Critical Section
fn correct3(map: &HashMap<i32, String>) {
    let data = {
        let guard = epoch::pin();
        map.get(&42, &guard).cloned()  // 복사
    };  // guard 해제

    // 오래 걸리는 작업
    process(data);
}
```

### 8.2 재진입 문제

```rust
// EBR은 재진입을 지원하지만 주의 필요

fn nested_pins() {
    let guard1 = epoch::pin();
    // ... 작업 ...

    {
        let guard2 = epoch::pin();  // 중첩 pin - OK
        // ...
    }  // guard2 drop되어도 guard1이 있어 안전

    // guard1 범위 내에서 작업 계속
}

// 문제: 서로 다른 스레드가 대기하는 경우
fn potential_deadlock() {
    let guard = epoch::pin();

    // 다른 스레드가 epoch 전이를 기다리는 동안
    // 이 스레드가 그 스레드를 기다리면 교착 상태

    // 해결: Critical Section을 짧게 유지
}
```

### 8.3 메모리 사용량 급증

```
EBR 메모리 사용량 문제:

시나리오: 하나의 스레드가 긴 Critical Section

Thread 0: ───────────────────────────────[pinned for long time]
Thread 1: [retire][retire][retire][retire][retire]...
Thread 2: [retire][retire][retire][retire][retire]...

결과: epoch 전이 불가 → 메모리 누적

해결책:
1. Critical Section 최소화
2. 주기적 flush (unpin 후 re-pin)
3. 메모리 제한 설정
```

## 9. 요약

### EBR 선택 기준

```
┌────────────────────────────────────────────────────────────────┐
│                  메모리 회수 방식 선택 가이드                    │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Critical Section이 짧은가?                                     │
│         │                                                       │
│    YES ─┼─ NO → Reference Counting 또는 HP                     │
│         │                                                       │
│         ▼                                                       │
│  구현 단순성 vs 최적 성능?                                      │
│         │                                                       │
│  단순 ─┼─ 최적                                                 │
│         │         │                                             │
│         ▼         ▼                                             │
│       EBR    Hazard Pointer                                    │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 핵심 비교표

| 특성 | EBR | Hazard Pointer | RCU |
|------|-----|----------------|-----|
| **복잡도** | 중간 | 높음 | 낮음 (Reader) |
| **Reader 비용** | 낮음 | 중간 | 매우 낮음 |
| **메모리 지연** | 2 epoch | 즉시 | Grace Period |
| **긴 CS** | 문제됨 | OK | OK (SRCU) |
| **Rust 지원** | Crossbeam | 직접 구현 | 없음 |

### 핵심 포인트

1. **장점**
   - Reader 오버헤드 낮음
   - 구현이 HP보다 단순
   - Crossbeam으로 쉽게 사용

2. **단점**
   - 해제 지연 (2 epoch)
   - 긴 Critical Section 시 문제
   - 메모리 사용량 예측 어려움

3. **최적 사용 사례**
   - Lock-Free 데이터 구조
   - 짧은 Critical Section
   - Rust 프로젝트 (Crossbeam)

EBR은 Hazard Pointer와 RCU 사이의 균형점으로, 특히 Rust 생태계에서 Crossbeam을 통해 널리 사용되는 검증된 메모리 회수 기법입니다.
