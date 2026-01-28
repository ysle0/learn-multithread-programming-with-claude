# Memory Allocator 내부 분석 (TCMalloc, jemalloc)

## 목차
1. [개요](#개요)
2. [TCMalloc](#tcmalloc)
3. [jemalloc](#jemalloc)
4. [mimalloc](#mimalloc)
5. [비교 분석](#비교-분석)
6. [Thread-Local Pool 구현 시사점](#thread-local-pool-구현-시사점)
7. [요약](#요약)

---

## 개요

고성능 메모리 할당기는 멀티스레드 환경에서의 확장성을 위해 **Thread-Local Caching**을 핵심 기법으로 사용합니다. 이 문서에서는 TCMalloc, jemalloc, mimalloc의 내부 구조를 분석하고, Thread-Local Pool 구현에 적용할 수 있는 기법을 정리합니다.

### 왜 기본 malloc이 느린가?

```
┌─────────────────────────────────────────────────────────────────┐
│                    glibc malloc 문제점                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  전역 락 병목:                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Thread 1 ──► malloc() ──┐                               │   │
│  │ Thread 2 ──► malloc() ──┼──► Global Lock ──► Allocate   │   │
│  │ Thread 3 ──► malloc() ──┤         ↑                     │   │
│  │ Thread 4 ──► malloc() ──┘     대기 시간!                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  glibc ptmalloc2 개선:                                           │
│  - Arena (힙 영역)를 여러 개 사용                                │
│  - 하지만 스레드 수 >> Arena 수이면 여전히 경합                   │
│  - 기본 Arena 수 = 8 * CPU 코어 수                               │
│                                                                 │
│  결과:                                                           │
│  - 높은 스레드 수에서 확장성 저하                                │
│  - 메모리 단편화                                                 │
│  - False Sharing                                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 해결책: Thread-Local Caching

```
┌─────────────────────────────────────────────────────────────────┐
│              Thread-Local Caching 아키텍처                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread 1        Thread 2        Thread 3        Thread 4       │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ Thread  │    │ Thread  │    │ Thread  │    │ Thread  │      │
│  │ Cache   │    │ Cache   │    │ Cache   │    │ Cache   │      │
│  │ (TLS)   │    │ (TLS)   │    │ (TLS)   │    │ (TLS)   │      │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘      │
│       │              │              │              │            │
│       │  Fast Path   │              │              │            │
│       │  (락 없음)   │              │              │            │
│       │              │              │              │            │
│       ▼              ▼              ▼              ▼            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Central Heap                          │   │
│  │               (락 필요, 하지만 드물게)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                       OS (mmap/sbrk)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  대부분의 할당/해제가 Thread Cache에서 처리됨 (락 없음!)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## TCMalloc

Google의 **Thread-Caching Malloc**은 Thread-Local Cache의 대표적 구현입니다.

### 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCMalloc Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    Front-End                               ││
│  │  ┌─────────────────────────────────────────────────────┐  ││
│  │  │ Per-Thread Cache (or Per-CPU Cache)                  │  ││
│  │  │                                                      │  ││
│  │  │  Size Class 0: [obj][obj][obj]...                    │  ││
│  │  │  Size Class 1: [obj][obj]...                         │  ││
│  │  │  Size Class 2: [obj][obj][obj][obj]...               │  ││
│  │  │  ...                                                 │  ││
│  │  │  Size Class N: [obj]...                              │  ││
│  │  └─────────────────────────────────────────────────────┘  ││
│  └────────────────────────────────────────────────────────────┘│
│                              │                                  │
│                              │ Overflow/Underflow               │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                    Middle-End                              ││
│  │  ┌─────────────────────────────────────────────────────┐  ││
│  │  │ Transfer Cache (Central Free List)                   │  ││
│  │  │                                                      │  ││
│  │  │  Thread Cache와 Central Heap 사이 버퍼               │  ││
│  │  │  배치 단위로 객체 이동                                │  ││
│  │  └─────────────────────────────────────────────────────┘  ││
│  └────────────────────────────────────────────────────────────┘│
│                              │                                  │
│                              ▼                                  │
│  ┌────────────────────────────────────────────────────────────┐│
│  │                     Back-End                               ││
│  │  ┌─────────────────────────────────────────────────────┐  ││
│  │  │ Page Heap (Span Allocator)                           │  ││
│  │  │                                                      │  ││
│  │  │  - Span: 연속된 페이지 그룹                          │  ││
│  │  │  - 큰 할당은 직접 Span에서 처리                      │  ││
│  │  │  - Page Map으로 Span 메타데이터 관리                 │  ││
│  │  └─────────────────────────────────────────────────────┘  ││
│  └────────────────────────────────────────────────────────────┘│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Size Classes

TCMalloc은 객체 크기를 **Size Class**로 분류합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCMalloc Size Classes                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Size Class │ Object Size │ Page Count │ Objects per Span      │
│  ───────────│─────────────│────────────│──────────────────      │
│      0      │     8       │     1      │    512                 │
│      1      │    16       │     1      │    256                 │
│      2      │    32       │     1      │    128                 │
│      3      │    48       │     1      │     85                 │
│      4      │    64       │     1      │     64                 │
│      5      │    80       │     1      │     51                 │
│      ...    │    ...      │    ...     │    ...                 │
│     85      │  256KB      │    32      │      1                 │
│                                                                 │
│  설계 원칙:                                                      │
│  - 내부 단편화 최대 12.5%                                        │
│  - 자주 사용되는 크기에 최적화                                   │
│  - Span 내 객체 수가 효율적이도록                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Thread Cache 동작

```c
// TCMalloc Thread Cache 의사 코드

struct ThreadCache {
    FreeList list[kNumClasses];  // Size class별 free list
    size_t size;                  // 캐시된 총 바이트
};

__thread ThreadCache* thread_cache;

void* tcmalloc_allocate(size_t size) {
    if (size > kMaxSmallSize) {
        return allocate_large(size);  // 큰 객체는 Page Heap에서
    }

    size_t cl = SizeClass(size);  // 크기 → Size Class

    // Fast Path: Thread Cache에서 할당
    ThreadCache* tc = thread_cache;
    if (tc->list[cl].count > 0) {
        return tc->list[cl].Pop();  // 락 없음!
    }

    // Slow Path: Central에서 가져오기
    return FetchFromCentral(cl);
}

void tcmalloc_deallocate(void* ptr) {
    size_t cl = GetSizeClass(ptr);  // 포인터에서 Size Class 계산

    ThreadCache* tc = thread_cache;

    // Fast Path: Thread Cache에 반환
    tc->list[cl].Push(ptr);  // 락 없음!
    tc->size += ClassToSize(cl);

    // 캐시가 너무 크면 Central에 반환
    if (tc->size > kMaxCacheSize) {
        Scavenge();
    }
}
```

### Per-CPU Mode (TCMalloc 최신)

```
┌─────────────────────────────────────────────────────────────────┐
│                    Per-CPU Caching                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  기존 Per-Thread:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 스레드당 캐시 → 많은 스레드 = 많은 메모리 사용         │   │
│  │ - 스레드 간 작업 불균형 시 비효율                        │   │
│  │ - 컨텍스트 스위칭 시 캐시 재사용 가능                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  새로운 Per-CPU:                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - CPU당 캐시 → 코어 수만큼만 캐시                        │   │
│  │ - restartable sequences (rseq) 사용                     │   │
│  │ - 컨텍스트 스위칭 시에도 같은 CPU면 같은 캐시            │   │
│  │ - 메모리 효율 + 캐시 지역성 향상                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  CPU 0 Cache    CPU 1 Cache    CPU 2 Cache    CPU 3 Cache      │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐        │
│  │[objects]│   │[objects]│   │[objects]│   │[objects]│        │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘        │
│       ↑             ↑             ↑             ↑              │
│    Thread A      Thread B      Thread C      Thread D          │
│    Thread E      Thread F      Thread G      Thread H          │
│    ...           ...           ...           ...               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## jemalloc

Facebook에서 사용하는 **jemalloc**은 또 다른 고성능 할당기입니다.

### 전체 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                    jemalloc Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread 1        Thread 2        Thread 3        Thread 4       │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│  │ tcache  │    │ tcache  │    │ tcache  │    │ tcache  │      │
│  │ (TLS)   │    │ (TLS)   │    │ (TLS)   │    │ (TLS)   │      │
│  └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘      │
│       │              │              │              │            │
│       │              │              │              │            │
│       ▼              ▼              ▼              ▼            │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐   │
│  │   Arena 0  │ │   Arena 1  │ │   Arena 2  │ │   Arena 3  │   │
│  │            │ │            │ │            │ │            │   │
│  │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │   │
│  │ │  Bins  │ │ │ │  Bins  │ │ │ │  Bins  │ │ │ │  Bins  │ │   │
│  │ │ (slab) │ │ │ │ (slab) │ │ │ │ (slab) │ │ │ │ (slab) │ │   │
│  │ └────────┘ │ │ └────────┘ │ │ └────────┘ │ │ └────────┘ │   │
│  │            │ │            │ │            │ │            │   │
│  │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │ │ ┌────────┐ │   │
│  │ │ Large  │ │ │ │ Large  │ │ │ │ Large  │ │ │ │ Large  │ │   │
│  │ │Allocat.│ │ │ │Allocat.│ │ │ │Allocat.│ │ │ │Allocat.│ │   │
│  │ └────────┘ │ │ └────────┘ │ │ └────────┘ │ │ └────────┘ │   │
│  └────────────┘ └────────────┘ └────────────┘ └────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     Extent Allocator                     │   │
│  │              (Chunk/Page 관리, Huge 할당)                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 주요 개념

```
┌─────────────────────────────────────────────────────────────────┐
│                    jemalloc 핵심 개념                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Arena:                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 독립적인 힙 영역                                        │   │
│  │ - 스레드가 Arena에 바인딩됨 (락 경합 감소)               │   │
│  │ - 기본: 4 * CPU 코어 수                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. tcache (Thread Cache):                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 스레드별 캐시 (TLS)                                    │   │
│  │ - 작은 객체만 캐시                                       │   │
│  │ - GC로 주기적 정리                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. Bin (Slab):                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 같은 크기의 객체 집합                                  │   │
│  │ - Run: 연속된 페이지 그룹                                │   │
│  │ - Bitmap으로 할당 상태 관리                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. Extent:                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 연속된 메모리 영역                                     │   │
│  │ - Small/Large/Huge 구분                                  │   │
│  │ - Dirty/Muzzy/Clean 상태 관리                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Size Classes

```
┌─────────────────────────────────────────────────────────────────┐
│                    jemalloc Size Classes                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Small (tcache 가능):                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Tiny:    8, 16                                           │   │
│  │ Quantum: 32, 48, 64, 80, 96, 112, 128                   │   │
│  │ Sub-page: 160, 192, 224, 256, 320, 384, 448, 512, ...   │   │
│  │ ...up to 14KB                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Large (Arena 직접):                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 14KB ~ 2MB (페이지 단위)                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Huge (Extent Allocator):                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ > 2MB (Chunk 단위)                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  특징:                                                           │
│  - Size Class 간격이 더 촘촘 (낮은 단편화)                       │
│  - 2의 제곱 + 4 step (1.25x 증가)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Cross-Thread Deallocation

```
┌─────────────────────────────────────────────────────────────────┐
│                Cross-Thread Deallocation                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  문제: Thread A가 할당, Thread B가 해제                          │
│                                                                 │
│  jemalloc 해결책:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Thread A (할당)         Thread B (해제)                 │   │
│  │  ┌─────────────┐        ┌─────────────┐                 │   │
│  │  │   tcache    │        │   tcache    │                 │   │
│  │  └──────┬──────┘        └──────┬──────┘                 │   │
│  │         │                      │                         │   │
│  │         ▼                      │                         │   │
│  │  ┌─────────────┐              │                         │   │
│  │  │   Arena 0   │ ◄────────────┘                         │   │
│  │  │             │   (직접 Arena에 반환)                   │   │
│  │  │ ┌─────────┐ │                                         │   │
│  │  │ │Bin Lock │ │   ← 락 필요하지만 Arena별로 분산        │   │
│  │  │ └─────────┘ │                                         │   │
│  │  └─────────────┘                                         │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  최적화:                                                         │
│  - 같은 Arena로 해제 시도 (locality)                             │
│  - tcache flush는 배치로 처리                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## mimalloc

Microsoft의 **mimalloc**은 최신 설계를 적용한 경량 할당기입니다.

### 핵심 특징

```
┌─────────────────────────────────────────────────────────────────┐
│                    mimalloc 특징                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Free List Sharding:                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 페이지당 local free list + thread-shared free list    │   │
│  │ - Cross-thread 해제 시 shared list에 추가               │   │
│  │ - 주기적으로 local로 병합                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. 페이지 기반 할당:                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │          Page (64KB default)                             │   │
│  │ ┌────────────────────────────────────────────────────┐  │   │
│  │ │ Block │ Block │ Block │ Block │ ... │ Free │ Free  │  │   │
│  │ └────────────────────────────────────────────────────┘  │   │
│  │     ↓       ↓       ↓                                    │   │
│  │   Used    Used    Used                                   │   │
│  │                                                          │   │
│  │ Local Free List: [Block] → [Block] → NULL               │   │
│  │ Thread Free List: [Block] → NULL (다른 스레드가 해제)   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. Segment 구조:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │          Segment (4MB default)                           │   │
│  │ ┌────────┬────────┬────────┬────────┬────────────────┐  │   │
│  │ │ Meta   │ Page 0 │ Page 1 │ Page 2 │ ...            │  │   │
│  │ │ data   │ (64KB) │ (64KB) │ (64KB) │                │  │   │
│  │ └────────┴────────┴────────┴────────┴────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 장점:                                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 코드 단순 (~8K LOC)                                    │   │
│  │ - 빠른 할당 (~10ns)                                      │   │
│  │ - 낮은 메모리 오버헤드                                   │   │
│  │ - 우수한 캐시 지역성                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Thread-Free List 처리

```c
// mimalloc의 Cross-Thread Deallocation (의사 코드)

struct Page {
    Block* local_free;   // 로컬 스레드 전용 (락 없음)
    _Atomic(Block*) thread_free;  // 다른 스레드용 (lock-free)
};

// 같은 스레드에서 해제 (빠름)
void mi_free_local(Page* page, Block* block) {
    block->next = page->local_free;
    page->local_free = block;  // 락 없음!
}

// 다른 스레드에서 해제 (lock-free CAS)
void mi_free_thread(Page* page, Block* block) {
    Block* old_head;
    do {
        old_head = atomic_load(&page->thread_free);
        block->next = old_head;
    } while (!atomic_compare_exchange_weak(&page->thread_free, &old_head, block));
}

// 할당 시 thread_free를 local_free로 병합
Block* mi_malloc_from_page(Page* page) {
    // 먼저 local에서 시도
    if (page->local_free) {
        return pop_local(page);
    }

    // local이 비었으면 thread_free 가져오기
    Block* list = atomic_exchange(&page->thread_free, NULL);
    if (list) {
        page->local_free = list->next;
        return list;
    }

    return NULL;  // 페이지가 가득 참
}
```

---

## 비교 분석

### 성능 특성

```
┌─────────────────────────────────────────────────────────────────┐
│                    할당기 성능 비교                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  벤치마크 (상대적 성능, mimalloc = 1.0):                         │
│                                                                 │
│  Single Thread:                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ glibc    │███████████████████████│ 2.3x                  │   │
│  │ tcmalloc │████████████│ 1.2x                             │   │
│  │ jemalloc │█████████████│ 1.3x                            │   │
│  │ mimalloc │██████████│ 1.0x (기준)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Multi Thread (16 cores):                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ glibc    │█████████████████████████████│ 8.5x            │   │
│  │ tcmalloc │██████████│ 1.0x                               │   │
│  │ jemalloc │███████████│ 1.1x                              │   │
│  │ mimalloc │██████████│ 1.0x (기준)                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Memory Overhead:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ glibc    │██████████████████│ 높음 (단편화)              │   │
│  │ tcmalloc │████████████████│ 중간 (캐시 유지)             │   │
│  │ jemalloc │██████████████│ 낮음 (정교한 관리)             │   │
│  │ mimalloc │████████████│ 낮음                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 기법 비교

| 기법 | TCMalloc | jemalloc | mimalloc |
|------|----------|----------|----------|
| Thread Cache | Per-Thread/CPU | Per-Thread | Per-Thread (Page) |
| Central Cache | Transfer Cache | Arena | Segment |
| Size Classes | ~85 | ~200+ | ~75 |
| Cross-Thread | Central로 반환 | Arena로 반환 | Thread-Free List |
| 메타데이터 | PageMap | Radix Tree | Page 내 |
| 코드 크기 | ~30K LOC | ~50K LOC | ~8K LOC |

---

## Thread-Local Pool 구현 시사점

### 핵심 설계 원칙

```
┌─────────────────────────────────────────────────────────────────┐
│           Thread-Local Pool 설계 원칙                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. Fast Path 최적화:                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 99%의 연산이 락 없이 완료되도록                         │   │
│  │ - TLS를 통한 직접 접근                                   │   │
│  │ - 단순한 Free List (push/pop만)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. Size Class 사용:                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 크기별로 분류하여 단편화 감소                          │   │
│  │ - 자주 사용되는 크기에 최적화                            │   │
│  │ - 내부 단편화 vs 외부 단편화 균형                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. Batch 연산:                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - Central과 주고받을 때 여러 객체를 한 번에              │   │
│  │ - 락 획득 비용 분산                                      │   │
│  │ - 캐시 효율 향상                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. Cross-Thread Deallocation:                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 다른 스레드가 해제할 수 있음을 고려                    │   │
│  │ - mimalloc 방식: thread-free list (lock-free)           │   │
│  │ - 주기적으로 local로 병합                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  5. 캐시 크기 제한:                                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 스레드별 캐시 크기 상한 설정                           │   │
│  │ - 초과 시 Central에 반환 (scavenge)                      │   │
│  │ - 메모리 사용량과 성능 균형                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 간단한 Thread-Local Pool 구현

```c
#include <stdatomic.h>
#include <stdlib.h>
#include <stdint.h>

#define BLOCK_SIZE 64
#define LOCAL_CACHE_MAX 256
#define BATCH_SIZE 32

typedef struct Block {
    struct Block* next;
} Block;

typedef struct {
    Block* local_free;          // 로컬 free list
    _Atomic(Block*) thread_free; // 다른 스레드가 해제한 블록
    size_t local_count;
} ThreadLocalPool;

// TLS
__thread ThreadLocalPool* tl_pool = NULL;

// Central Pool (전역)
typedef struct {
    _Atomic(Block*) free_list;
    pthread_mutex_t lock;
} CentralPool;

CentralPool central_pool = {NULL, PTHREAD_MUTEX_INITIALIZER};

// 초기화
void pool_init_thread() {
    tl_pool = aligned_alloc(64, sizeof(ThreadLocalPool));
    tl_pool->local_free = NULL;
    atomic_store(&tl_pool->thread_free, NULL);
    tl_pool->local_count = 0;
}

// thread_free를 local로 병합
static void merge_thread_free() {
    Block* list = atomic_exchange(&tl_pool->thread_free, NULL);
    while (list) {
        Block* next = list->next;
        list->next = tl_pool->local_free;
        tl_pool->local_free = list;
        tl_pool->local_count++;
        list = next;
    }
}

// Central에서 배치로 가져오기
static void fetch_from_central() {
    pthread_mutex_lock(&central_pool.lock);

    for (int i = 0; i < BATCH_SIZE; i++) {
        Block* block = atomic_load(&central_pool.free_list);
        if (block == NULL) {
            // Central도 비었으면 새로 할당
            block = aligned_alloc(64, BLOCK_SIZE);
        } else {
            Block* next = block->next;
            atomic_store(&central_pool.free_list, next);
        }
        block->next = tl_pool->local_free;
        tl_pool->local_free = block;
        tl_pool->local_count++;
    }

    pthread_mutex_unlock(&central_pool.lock);
}

// Central에 배치로 반환
static void return_to_central() {
    pthread_mutex_lock(&central_pool.lock);

    for (int i = 0; i < BATCH_SIZE && tl_pool->local_count > LOCAL_CACHE_MAX / 2; i++) {
        Block* block = tl_pool->local_free;
        tl_pool->local_free = block->next;
        tl_pool->local_count--;

        block->next = atomic_load(&central_pool.free_list);
        atomic_store(&central_pool.free_list, block);
    }

    pthread_mutex_unlock(&central_pool.lock);
}

// 할당 (락 없음!)
void* pool_alloc() {
    // thread_free 확인
    if (tl_pool->local_free == NULL) {
        merge_thread_free();
    }

    // local에서 할당
    if (tl_pool->local_free) {
        Block* block = tl_pool->local_free;
        tl_pool->local_free = block->next;
        tl_pool->local_count--;
        return block;
    }

    // Central에서 가져오기
    fetch_from_central();
    return pool_alloc();
}

// 해제
void pool_free(void* ptr, ThreadLocalPool* owner) {
    Block* block = (Block*)ptr;

    if (owner == tl_pool) {
        // 같은 스레드: local에 반환
        block->next = tl_pool->local_free;
        tl_pool->local_free = block;
        tl_pool->local_count++;

        // 캐시가 너무 크면 Central에 반환
        if (tl_pool->local_count > LOCAL_CACHE_MAX) {
            return_to_central();
        }
    } else {
        // 다른 스레드: owner의 thread_free에 반환 (lock-free)
        Block* old_head;
        do {
            old_head = atomic_load(&owner->thread_free);
            block->next = old_head;
        } while (!atomic_compare_exchange_weak(&owner->thread_free, &old_head, block));
    }
}
```

---

## 요약

### 핵심 기법 정리

| 기법 | 목적 | 적용 |
|------|------|------|
| Thread-Local Cache | 락 제거 | TLS + Free List |
| Size Classes | 단편화 감소 | 크기별 풀 분리 |
| Batch Transfer | 락 비용 분산 | 여러 객체 한번에 |
| Free List Sharding | Cross-thread 처리 | local + thread-free |
| Cache Line Alignment | False Sharing 방지 | 64B 정렬 |

### 설계 선택 가이드

```
┌─────────────────────────────────────────────────────────────────┐
│              Thread-Local Pool 설계 선택                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  단순함 우선 (대부분의 경우):                                    │
│  └── mimalloc 스타일 (local + thread-free list)                │
│                                                                 │
│  높은 스레드 수:                                                 │
│  └── TCMalloc Per-CPU 스타일                                   │
│                                                                 │
│  메모리 효율 중요:                                               │
│  └── jemalloc 스타일 (촘촘한 Size Class)                       │
│                                                                 │
│  Cross-thread 해제 빈번:                                         │
│  └── mimalloc thread-free list 또는 Arena 기반               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 관련 문서

- [Thread Local Storage](../01-fundamentals/05-thread-local-storage.md) - TLS 기반
- [False Sharing](../09-appendix/false-sharing.md) - 캐시 라인 정렬
- [Lock-Free Stack](../05-lock-free-programming/03-lock-free-stack.md) - Free List 구현
- [CAS 연산](../05-lock-free-programming/01-cas-operation.md) - Lock-free 기법

---

## 참고 자료

- [TCMalloc Design](https://google.github.io/tcmalloc/design.html)
- [jemalloc - A Scalable Concurrent malloc](https://jemalloc.net/)
- [mimalloc - A Compact General Purpose Allocator](https://microsoft.com/mimalloc)
- [Understanding glibc malloc](https://sploitfun.wordpress.com/2015/02/10/understanding-glibc-malloc/)
