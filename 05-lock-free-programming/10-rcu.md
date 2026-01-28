# RCU (Read-Copy-Update)

## 1. RCU 개요

### 1.1 RCU란?

RCU(Read-Copy-Update)는 Linux 커널에서 개발된 동기화 메커니즘으로, **읽기 작업이 압도적으로 많은 워크로드**에서 극도로 높은 성능을 제공합니다.

```
전통적 락 vs RCU 비교:

[Reader-Writer Lock]
Reader 1: ----[LOCK]----[READ]----[UNLOCK]----
Reader 2:               [WAIT]----[LOCK]----[READ]----[UNLOCK]
Writer:                                      [WAIT]----[LOCK]----[WRITE]

[RCU]
Reader 1: ----[READ - NO LOCK!]----
Reader 2: ----[READ - NO LOCK!]----
Writer:   ----[COPY]----[UPDATE]----[WAIT FOR READERS]----[FREE OLD]
```

### 1.2 핵심 원리

RCU의 핵심 아이디어:
1. **Reader는 락 없이 읽기**: 오버헤드 거의 제로
2. **Writer는 복사 후 업데이트**: Copy-on-Write 방식
3. **Grace Period**: 모든 기존 Reader가 완료될 때까지 대기
4. **지연된 해제**: 안전한 시점에 구버전 메모리 해제

```c
// RCU의 기본 동작 흐름
struct data {
    int value;
    struct rcu_head rcu;
};

struct data __rcu *global_ptr;

// Reader - 락 없이 읽기
void reader(void)
{
    struct data *p;

    rcu_read_lock();           // Preemption 비활성화만 (락 아님!)
    p = rcu_dereference(global_ptr);  // 포인터 안전하게 읽기
    if (p) {
        printf("value: %d\n", p->value);
    }
    rcu_read_unlock();         // Preemption 재활성화
}

// Writer - 복사, 업데이트, 지연 해제
void writer(int new_value)
{
    struct data *old, *new;

    new = kmalloc(sizeof(*new), GFP_KERNEL);
    new->value = new_value;

    old = rcu_dereference_protected(global_ptr, lockdep_is_held(&update_lock));
    rcu_assign_pointer(global_ptr, new);  // 원자적 포인터 업데이트

    synchronize_rcu();         // Grace Period 대기
    // 또는 call_rcu(&old->rcu, callback);  // 비동기 해제

    kfree(old);                // 이제 안전하게 해제
}
```

## 2. RCU 내부 구조

### 2.1 Grace Period

Grace Period는 RCU의 핵심 개념입니다. **모든 기존 Reader가 Critical Section을 벗어날 때까지의 기간**입니다.

```
시간 흐름 →

CPU 0 (Reader):  [---READ---]
CPU 1 (Reader):       [---READ---]
CPU 2 (Reader):            [---READ---]
Writer:          [UPDATE]
                         |←-- Grace Period --→|
                                              [FREE OLD - 안전!]

Grace Period 동안:
- 새 Reader는 새 버전만 볼 수 있음
- 기존 Reader는 구버전 또는 신버전을 봄
- Grace Period 이후 구버전을 보는 Reader 없음
```

### 2.2 Quiescent State

각 CPU가 "RCU Critical Section에 없다"는 것을 나타내는 상태입니다.

```c
// Quiescent State 예시
void quiescent_states(void)
{
    // 다음 상황에서 CPU는 Quiescent State:

    // 1. 컨텍스트 스위치 발생 시
    schedule();  // Quiescent State!

    // 2. User space로 전환 시
    return_to_userspace();  // Quiescent State!

    // 3. Idle 상태 진입 시
    cpu_idle();  // Quiescent State!

    // 4. rcu_read_unlock() 호출 후
    rcu_read_unlock();  // Quiescent State 보고 가능
}
```

### 2.3 RCU 상태 머신

```
┌─────────────────────────────────────────────────────────────┐
│                    RCU Grace Period FSM                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   [IDLE] ──(rcu_assign_pointer)──→ [GP_REQUESTED]           │
│                                            │                 │
│                                            ▼                 │
│                                    [GP_STARTED]              │
│                                            │                 │
│              ┌─────────────────────────────┼─────────┐       │
│              ▼                             ▼         ▼       │
│         [CPU 0]                       [CPU 1]   [CPU N]      │
│     Wait for QS                   Wait for QS   Wait for QS  │
│              │                             │         │       │
│              └─────────────────────────────┼─────────┘       │
│                                            ▼                 │
│                             (All CPUs reported QS)           │
│                                            │                 │
│                                            ▼                 │
│                                    [GP_COMPLETED]            │
│                                            │                 │
│   [IDLE] ←──(callbacks invoked)───────────┘                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## 3. RCU API 상세

### 3.1 Reader API

```c
// Linux 커널 RCU Reader API

// Critical Section 진입
rcu_read_lock();
// - Preemption 비활성화 (CONFIG_PREEMPT_RCU가 아닌 경우)
// - SRCU의 경우 다른 메커니즘 사용

// 포인터 안전하게 읽기
struct foo *p = rcu_dereference(global_ptr);
// - 컴파일러 최적화 방지
// - 필요한 메모리 배리어 삽입 (Alpha에서 중요)
// - __rcu 포인터 annotation 검증

// Critical Section 탈출
rcu_read_unlock();
// - Preemption 재활성화

// 사용 예시
void safe_read(void)
{
    struct foo *p;

    rcu_read_lock();

    p = rcu_dereference(global_ptr);
    if (p) {
        // p 사용 - Critical Section 내에서만!
        do_something(p);
    }

    rcu_read_unlock();
    // 여기서 p 사용 금지! (이미 해제되었을 수 있음)
}
```

### 3.2 Writer API

```c
// Linux 커널 RCU Writer API

// 포인터 원자적 업데이트
rcu_assign_pointer(global_ptr, new_ptr);
// - 필요한 메모리 배리어 삽입
// - 이전의 모든 초기화가 완료됨을 보장

// 동기 방식: Grace Period 완료까지 블록
synchronize_rcu();
// - 현재 진행 중인 모든 Reader가 완료될 때까지 대기
// - 오래 걸릴 수 있음 (수 밀리초)

// 비동기 방식: Callback 등록
void my_callback(struct rcu_head *head)
{
    struct foo *p = container_of(head, struct foo, rcu);
    kfree(p);
}

call_rcu(&old_ptr->rcu, my_callback);
// - Grace Period 후 callback 호출
// - Writer가 블록되지 않음

// RCU 배리어: 모든 callback 완료 대기
rcu_barrier();
// - 모듈 언로드 시 필수
// - 등록된 모든 callback이 실행됨을 보장
```

### 3.3 리스트 순회 API

```c
// RCU-protected 리스트 순회
#include <linux/rculist.h>

struct my_node {
    int data;
    struct list_head list;
    struct rcu_head rcu;
};

LIST_HEAD(my_list);
DEFINE_SPINLOCK(list_lock);

// Reader: RCU로 보호된 순회
void list_reader(void)
{
    struct my_node *node;

    rcu_read_lock();
    list_for_each_entry_rcu(node, &my_list, list) {
        process(node->data);
    }
    rcu_read_unlock();
}

// Writer: 노드 추가
void list_add_node(struct my_node *new)
{
    spin_lock(&list_lock);
    list_add_rcu(&new->list, &my_list);
    spin_unlock(&list_lock);
}

// Writer: 노드 삭제
void list_del_node(struct my_node *node)
{
    spin_lock(&list_lock);
    list_del_rcu(&node->list);
    spin_unlock(&list_lock);

    call_rcu(&node->rcu, free_node_callback);
}
```

## 4. RCU 변형들

### 4.1 Sleepable RCU (SRCU)

일반 RCU는 Critical Section에서 sleep 불가. SRCU는 이를 허용합니다.

```c
#include <linux/srcu.h>

DEFINE_SRCU(my_srcu);

// SRCU Reader
void srcu_reader(void)
{
    int idx;

    idx = srcu_read_lock(&my_srcu);

    // Sleep 가능!
    might_sleep();
    do_blocking_operation();

    srcu_read_unlock(&my_srcu, idx);
}

// SRCU Writer
void srcu_writer(void)
{
    // ... update ...

    synchronize_srcu(&my_srcu);
    // 또는
    call_srcu(&my_srcu, &node->rcu, callback);
}
```

```
SRCU vs Classic RCU 비교:

┌─────────────────┬───────────────────┬──────────────────┐
│     특성        │    Classic RCU    │      SRCU        │
├─────────────────┼───────────────────┼──────────────────┤
│ Sleep in CS     │        ✗          │        ✓         │
│ Reader 오버헤드 │      최소         │      약간        │
│ Grace Period    │      빠름         │      느림        │
│ 메모리 사용     │      적음         │      많음        │
│ 사용 케이스     │ 대부분의 경우     │ Blocking I/O    │
└─────────────────┴───────────────────┴──────────────────┘
```

### 4.2 Tasks RCU

태스크의 voluntary context switch를 Quiescent State로 간주합니다.

```c
// Tasks RCU - trampoline, BPF 등에서 사용
synchronize_rcu_tasks();

// Rude variant - preemption point를 QS로 간주
synchronize_rcu_tasks_rude();

// Trace variant - tracing에 특화
synchronize_rcu_tasks_trace();
```

### 4.3 Tree RCU

대규모 시스템(수백~수천 CPU)을 위한 확장 가능한 RCU 구현입니다.

```
Tree RCU 구조 (64 CPU 예시):

                    [Root Node]
                    Grace Period
                    Controller
                   /          \
           [Node 0]            [Node 1]
          /   |   \           /   |   \
      [Leaf] [Leaf] [Leaf]  [Leaf] [Leaf] [Leaf]
       0-15   16-31  32-47   48-55  56-63  ...
        │      │      │       │      │
     CPUs 0-15 ...   ...     ...    CPUs 56-63

특징:
- O(log N) Grace Period 확인
- NUMA 지역성 최적화
- CPU 그룹별 Quiescent State 집계
```

## 5. User-space RCU (URCU)

### 5.1 URCU 라이브러리

```c
#include <urcu.h>
#include <urcu/rculfhash.h>  // RCU Lock-Free Hash Table

// URCU 초기화 (스레드별로 호출)
rcu_register_thread();

// Reader
void urcu_reader(void)
{
    struct my_data *p;

    rcu_read_lock();
    p = rcu_dereference(global_ptr);
    if (p) {
        process(p);
    }
    rcu_read_unlock();
}

// Writer
void urcu_writer(struct my_data *new_data)
{
    struct my_data *old;

    old = rcu_xchg_pointer(&global_ptr, new_data);

    synchronize_rcu();  // 또는 call_rcu()

    free(old);
}

// 종료
rcu_unregister_thread();
```

### 5.2 URCU Flavors

```c
// 1. General Purpose (기본)
#include <urcu.h>
// - Signal 기반 Grace Period 감지
// - 대부분의 경우 적합

// 2. QSBR (Quiescent State Based Reclamation)
#include <urcu-qsbr.h>
// - Reader가 명시적으로 QS 보고
// - 가장 빠른 read-side
void reader_qsbr(void)
{
    // 주기적으로 호출 필요
    rcu_quiescent_state();
}

// 3. Membarrier
#include <urcu-memb.h>
// - membarrier() 시스템 콜 사용
// - Signal 없이 동작

// 4. Bullet-Proof
#include <urcu-bp.h>
// - 라이브러리 등록 없이 동작
// - fork() 안전
```

### 5.3 URCU 기반 해시 테이블

```c
#include <urcu.h>
#include <urcu/rculfhash.h>

struct my_node {
    int key;
    int value;
    struct cds_lfht_node ht_node;
    struct rcu_head rcu_head;
};

// 해시 테이블 생성
struct cds_lfht *ht = cds_lfht_new(
    1,      // 초기 크기
    1,      // 최소 크기
    0,      // 최대 크기 (0 = 무제한)
    CDS_LFHT_AUTO_RESIZE,
    NULL
);

// 조회
struct my_node *lookup(int key)
{
    struct cds_lfht_iter iter;
    struct cds_lfht_node *ht_node;
    struct my_node *node = NULL;

    rcu_read_lock();
    cds_lfht_lookup(ht, hash_fn(key), match_fn, &key, &iter);
    ht_node = cds_lfht_iter_get_node(&iter);
    if (ht_node) {
        node = container_of(ht_node, struct my_node, ht_node);
    }
    rcu_read_unlock();

    return node;
}

// 삽입
void insert(struct my_node *node)
{
    rcu_read_lock();
    cds_lfht_add(ht, hash_fn(node->key), &node->ht_node);
    rcu_read_unlock();
}

// 삭제
void delete(struct my_node *node)
{
    rcu_read_lock();
    int ret = cds_lfht_del(ht, &node->ht_node);
    rcu_read_unlock();

    if (!ret) {
        call_rcu(&node->rcu_head, free_node);
    }
}
```

## 6. RCU 구현 심층 분석

### 6.1 간단한 RCU 구현

```c
// 교육용 단순화된 RCU 구현
#include <stdatomic.h>
#include <pthread.h>

#define MAX_THREADS 64

// 각 스레드의 RCU 상태
struct rcu_reader {
    _Alignas(64) atomic_uint_fast64_t counter;  // 홀수: Critical Section, 짝수: 밖
};

static struct rcu_reader readers[MAX_THREADS];
static atomic_uint_fast64_t global_counter = 0;
static __thread int thread_id = -1;

void rcu_register_thread(int id)
{
    thread_id = id;
    atomic_store(&readers[id].counter, 0);
}

void rcu_read_lock(void)
{
    // 현재 global_counter 값을 홀수로 저장 (Critical Section 진입)
    uint64_t gc = atomic_load_explicit(&global_counter, memory_order_acquire);
    atomic_store_explicit(&readers[thread_id].counter, gc | 1, memory_order_release);
    atomic_thread_fence(memory_order_seq_cst);
}

void rcu_read_unlock(void)
{
    // 짝수로 변경 (Critical Section 탈출)
    atomic_store_explicit(&readers[thread_id].counter, 0, memory_order_release);
}

void synchronize_rcu(void)
{
    // Grace Period 시작
    uint64_t gc = atomic_fetch_add(&global_counter, 2) + 2;

    // 모든 Reader가 새로운 Grace Period로 전환될 때까지 대기
    for (int i = 0; i < MAX_THREADS; i++) {
        uint64_t reader_counter;
        do {
            reader_counter = atomic_load_explicit(&readers[i].counter,
                                                   memory_order_acquire);
            // Reader가 Critical Section 밖이거나 (짝수)
            // 새로운 Grace Period에서 시작했으면 (>= gc) 통과
        } while ((reader_counter & 1) && (reader_counter < gc));
    }
}
```

### 6.2 QSBR 구현

```c
// Quiescent State Based Reclamation
#include <stdatomic.h>

#define MAX_THREADS 64

struct qsbr_state {
    _Alignas(64) atomic_uint_fast64_t counter;
};

static struct qsbr_state qsbr[MAX_THREADS];
static atomic_uint_fast64_t global_epoch = 0;
static __thread int my_id;

void qsbr_register(int id)
{
    my_id = id;
    atomic_store(&qsbr[id].counter, 0);
}

// Reader는 이 함수를 주기적으로 호출 (예: 이벤트 루프에서)
void rcu_quiescent_state(void)
{
    uint64_t epoch = atomic_load_explicit(&global_epoch, memory_order_acquire);
    atomic_store_explicit(&qsbr[my_id].counter, epoch, memory_order_release);
}

// 오프라인 상태 (스레드가 RCU 데이터 접근 안 함)
void rcu_thread_offline(void)
{
    atomic_store_explicit(&qsbr[my_id].counter, UINT64_MAX, memory_order_release);
}

void rcu_thread_online(void)
{
    uint64_t epoch = atomic_load(&global_epoch);
    atomic_store_explicit(&qsbr[my_id].counter, epoch, memory_order_release);
}

void synchronize_rcu_qsbr(void)
{
    uint64_t target = atomic_fetch_add(&global_epoch, 1) + 1;

    // 모든 스레드가 새 epoch를 확인할 때까지 대기
    for (int i = 0; i < MAX_THREADS; i++) {
        uint64_t c;
        do {
            c = atomic_load_explicit(&qsbr[i].counter, memory_order_acquire);
            // UINT64_MAX = 오프라인, >= target = 새 epoch 확인
        } while (c != UINT64_MAX && c < target);
    }
}
```

## 7. RCU 사용 패턴

### 7.1 NBS (Non-Blocking Synchronization) with RCU

```c
// RCU + Reference Counting 조합
struct ref_counted {
    atomic_int refcount;
    struct rcu_head rcu;
    // data...
};

struct ref_counted *acquire_ref(void)
{
    struct ref_counted *p;

    rcu_read_lock();
    p = rcu_dereference(global_ptr);
    if (p) {
        // RCU 보호 하에 참조 카운트 증가
        if (atomic_fetch_add(&p->refcount, 1) <= 0) {
            // 이미 해제 진행 중
            atomic_fetch_sub(&p->refcount, 1);
            p = NULL;
        }
    }
    rcu_read_unlock();

    return p;
}

void release_ref(struct ref_counted *p)
{
    if (atomic_fetch_sub(&p->refcount, 1) == 1) {
        call_rcu(&p->rcu, free_callback);
    }
}
```

### 7.2 Publish-Subscribe 패턴

```c
// 여러 필드를 원자적으로 업데이트
struct config {
    int timeout;
    int retries;
    char *server;
    // 많은 필드들...
};

struct config __rcu *current_config;

// Publisher: 전체 config를 원자적으로 교체
void update_config(int timeout, int retries, const char *server)
{
    struct config *new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);
    struct config *old_cfg;

    new_cfg->timeout = timeout;
    new_cfg->retries = retries;
    new_cfg->server = kstrdup(server, GFP_KERNEL);

    old_cfg = rcu_replace_pointer(current_config, new_cfg,
                                   lockdep_is_held(&config_lock));

    call_rcu(&old_cfg->rcu, free_config);
}

// Subscriber: 일관된 config 스냅샷 읽기
void use_config(void)
{
    struct config *cfg;

    rcu_read_lock();
    cfg = rcu_dereference(current_config);

    // cfg의 모든 필드가 일관성 있게 보임
    connect_with_timeout(cfg->server, cfg->timeout);

    rcu_read_unlock();
}
```

### 7.3 Existence Guarantee 패턴

```c
// RCU가 객체의 존재를 보장
struct connection {
    int fd;
    struct sockaddr addr;
    struct rcu_head rcu;
};

struct connection __rcu *connections[MAX_CONNECTIONS];

// 검색: RCU가 connection 존재 보장
void send_to_connection(int idx, void *data, size_t len)
{
    struct connection *conn;

    rcu_read_lock();
    conn = rcu_dereference(connections[idx]);
    if (conn) {
        // conn이 존재함이 보장됨
        send(conn->fd, data, len, 0);
    }
    rcu_read_unlock();
}

// 삭제: Grace Period 후 안전하게 해제
void close_connection(int idx)
{
    struct connection *conn;

    spin_lock(&conn_lock);
    conn = rcu_dereference_protected(connections[idx],
                                     lockdep_is_held(&conn_lock));
    rcu_assign_pointer(connections[idx], NULL);
    spin_unlock(&conn_lock);

    if (conn) {
        close(conn->fd);
        call_rcu(&conn->rcu, free_connection);
    }
}
```

## 8. RCU 성능 분석

### 8.1 벤치마크 비교

```
읽기 위주 워크로드 (99% Read, 1% Write):

Operations/sec (높을수록 좋음)
│
│  RCU
│  ████████████████████████████████████████████  4,500,000
│
│  RW Lock
│  ██████████████████████  2,200,000
│
│  Mutex
│  ███████████  1,100,000
│
└────────────────────────────────────────────────────────

CPU 수 증가에 따른 확장성:

Throughput
│           RCU (거의 선형)
│          ╱
│         ╱    RW Lock
│        ╱    ╱
│       ╱    ╱
│      ╱    ╱     Mutex (포화)
│     ╱    ╱     ─────────────
│    ╱    ╱
│   ╱    ╱
│  ╱   ╱
│ ╱  ╱
│╱ ╱
└──────────────────────────────────────→ CPUs
 1  2  4  8  16  32  64
```

### 8.2 Reader 오버헤드 비교

```c
// 각 동기화 방식의 Reader 비용

// Mutex: ~100 cycles (contention 시 훨씬 더)
pthread_mutex_lock(&mutex);
data = shared_data;
pthread_mutex_unlock(&mutex);

// RW Lock: ~50-80 cycles
pthread_rwlock_rdlock(&rwlock);
data = shared_data;
pthread_rwlock_rdunlock(&rwlock);

// RCU: ~5-10 cycles (메모리 배리어만)
rcu_read_lock();   // local operation only
data = rcu_dereference(ptr);
rcu_read_unlock(); // local operation only
```

### 8.3 Grace Period 레이턴시

```
Grace Period 레이턴시 분포 (마이크로초):

synchronize_rcu() 호출 시:

     빈도
       │
       │    ╭─╮
       │   ╱   ╲
       │  ╱     ╲
       │ ╱       ╲
       │╱         ╲
       │           ╲
       │            ╲
       └─────────────────────────────→ 레이턴시 (μs)
          10   50   100  200  500

평균: ~50-100μs
최대: 수 밀리초 (시스템 부하에 따라)

call_rcu() 사용 시 Writer는 블록 안됨
```

## 9. RCU 주의사항

### 9.1 일반적인 실수

```c
// 실수 1: Critical Section 밖에서 포인터 사용
void bug1(void)
{
    struct data *p;

    rcu_read_lock();
    p = rcu_dereference(global_ptr);
    rcu_read_unlock();

    // BUG! p가 이미 해제되었을 수 있음
    use(p);
}

// 실수 2: rcu_dereference 없이 포인터 읽기
void bug2(void)
{
    rcu_read_lock();
    struct data *p = global_ptr;  // BUG! 컴파일러 최적화 문제
    use(p);
    rcu_read_unlock();
}

// 실수 3: Writer 동기화 없이 업데이트
void bug3(struct data *new_data)
{
    // BUG! 여러 Writer가 동시에 업데이트하면 문제
    rcu_assign_pointer(global_ptr, new_data);
}

// 올바른 방법
DEFINE_SPINLOCK(update_lock);

void correct3(struct data *new_data)
{
    spin_lock(&update_lock);
    rcu_assign_pointer(global_ptr, new_data);
    spin_unlock(&update_lock);
}

// 실수 4: synchronize_rcu 없이 메모리 해제
void bug4(void)
{
    struct data *old = rcu_dereference(global_ptr);
    rcu_assign_pointer(global_ptr, new_data);
    kfree(old);  // BUG! Reader가 아직 사용 중일 수 있음
}
```

### 9.2 RCU 사용이 부적절한 경우

```c
// 1. 쓰기 빈도가 높은 경우
// Grace Period 오버헤드가 누적됨

// 2. 긴 Critical Section
void bad_for_rcu(void)
{
    rcu_read_lock();

    // 오래 걸리는 작업...
    slow_operation();  // Grace Period 지연

    rcu_read_unlock();
}

// 3. Reader에서 Sleep이 필요한 경우 (SRCU 사용)
void needs_sleep(void)
{
    rcu_read_lock();
    blocking_io();  // BUG in classic RCU!
    rcu_read_unlock();
}

// 4. 강한 일관성이 필요한 경우
// RCU는 Eventual Consistency 제공
```

## 10. 실전 적용 예시

### 10.1 라우팅 테이블

```c
// Linux 커널 스타일 라우팅 테이블
struct route_entry {
    __be32 dest;
    __be32 gateway;
    int metric;
    struct hlist_node node;
    struct rcu_head rcu;
};

#define ROUTE_HASH_SIZE 256
struct hlist_head route_table[ROUTE_HASH_SIZE];
DEFINE_SPINLOCK(route_lock);

// 패킷 포워딩 (매우 빈번히 호출)
struct route_entry *lookup_route(__be32 dest)
{
    struct route_entry *rt;
    u32 hash = jhash_1word(dest, 0) % ROUTE_HASH_SIZE;

    rcu_read_lock();
    hlist_for_each_entry_rcu(rt, &route_table[hash], node) {
        if (rt->dest == dest) {
            rcu_read_unlock();
            return rt;  // 주의: 사용자가 RCU 또는 refcount로 보호
        }
    }
    rcu_read_unlock();

    return NULL;
}

// 라우트 추가 (드물게 호출)
int add_route(__be32 dest, __be32 gateway, int metric)
{
    struct route_entry *rt;
    u32 hash = jhash_1word(dest, 0) % ROUTE_HASH_SIZE;

    rt = kmalloc(sizeof(*rt), GFP_KERNEL);
    if (!rt)
        return -ENOMEM;

    rt->dest = dest;
    rt->gateway = gateway;
    rt->metric = metric;

    spin_lock(&route_lock);
    hlist_add_head_rcu(&rt->node, &route_table[hash]);
    spin_unlock(&route_lock);

    return 0;
}

// 라우트 삭제
void del_route(struct route_entry *rt)
{
    spin_lock(&route_lock);
    hlist_del_rcu(&rt->node);
    spin_unlock(&route_lock);

    call_rcu(&rt->rcu, free_route_callback);
}
```

### 10.2 연결 추적 테이블

```c
// 네트워크 연결 추적 (conntrack)
struct conn_entry {
    struct nf_conntrack_tuple tuple;
    enum ip_conntrack_status status;
    atomic_t refcount;
    struct hlist_node hnode;
    struct rcu_head rcu;
};

// 패킷마다 호출되는 빠른 경로
struct conn_entry *find_connection(const struct sk_buff *skb)
{
    struct conn_entry *ct;
    u32 hash = nf_conntrack_hash(&tuple);

    rcu_read_lock();
    hlist_for_each_entry_rcu(ct, &conntrack_hash[hash], hnode) {
        if (nf_ct_tuple_equal(&ct->tuple, &tuple)) {
            // RCU + refcount 조합
            if (atomic_inc_not_zero(&ct->refcount)) {
                rcu_read_unlock();
                return ct;
            }
        }
    }
    rcu_read_unlock();

    return NULL;
}
```

## 11. 요약

### RCU 선택 기준

```
┌────────────────────────────────────────────────────────────────┐
│                     RCU 사용 결정 트리                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  읽기/쓰기 비율이 높은가? (>90% 읽기)                           │
│         │                                                       │
│    YES ─┼─ NO → Mutex/RW Lock 고려                             │
│         │                                                       │
│         ▼                                                       │
│  Reader에서 Sleep이 필요한가?                                   │
│         │                                                       │
│    YES ─┼─ NO → Classic RCU 사용                               │
│         │                                                       │
│         ▼                                                       │
│  SRCU 사용                                                      │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

### 핵심 포인트

| 특성 | 설명 |
|------|------|
| **강점** | 읽기 오버헤드 최소화, 뛰어난 확장성 |
| **약점** | Grace Period 레이턴시, 메모리 사용량 |
| **적합한 경우** | 읽기 위주 워크로드, 많은 CPU |
| **부적합한 경우** | 쓰기 위주, 강한 일관성 필요 |
| **주요 사용처** | Linux 커널, 라우팅, 연결 추적 |

RCU는 읽기가 압도적으로 많은 상황에서 다른 어떤 동기화 방법보다 뛰어난 성능을 제공하며, 특히 Linux 커널에서 광범위하게 사용되는 검증된 기술입니다.
