# False Sharing

## 목차
1. [개요](#개요)
2. [캐시 라인의 이해](#캐시-라인의-이해)
3. [False Sharing 발생 조건](#false-sharing-발생-조건)
4. [성능 영향](#성능-영향)
5. [탐지 방법](#탐지-방법)
6. [해결 방법](#해결-방법)
7. [실전 예제](#실전-예제)
8. [요약](#요약)

---

## 개요

**False Sharing**은 서로 다른 데이터지만 같은 캐시 라인에 위치하여 발생하는 성능 저하 현상입니다. 논리적으로 공유하지 않는 데이터가 물리적으로 캐시를 공유하여 불필요한 캐시 무효화가 발생합니다.

### 핵심 문제

```
┌─────────────────────────────────────────────────────────────────┐
│                    False Sharing 개요                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  메모리 레이아웃:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │        Cache Line (64 bytes)                             │   │
│  │ ┌──────────┬──────────┬──────────┬──────────┬─────────┐ │   │
│  │ │ counter_A│ counter_B│ counter_C│ counter_D│ padding │ │   │
│  │ │ (8 bytes)│ (8 bytes)│ (8 bytes)│ (8 bytes)│         │ │   │
│  │ └──────────┴──────────┴──────────┴──────────┴─────────┘ │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Thread 1: counter_A++                                          │
│  Thread 2: counter_B++                                          │
│  Thread 3: counter_C++                                          │
│  Thread 4: counter_D++                                          │
│                                                                 │
│  문제:                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ - 각 스레드는 독립적인 카운터를 사용                      │   │
│  │ - 하지만 모두 같은 캐시 라인에 있음!                      │   │
│  │ - 한 스레드가 쓰면 다른 모든 코어의 캐시가 무효화됨       │   │
│  │ - 결과: 캐시 미스 폭증, 성능 10-100배 저하 가능          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 캐시 라인의 이해

### CPU 캐시 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                    CPU 캐시 계층                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CPU Core 0          CPU Core 1          CPU Core 2            │
│  ┌─────────┐         ┌─────────┐         ┌─────────┐           │
│  │ Register│         │ Register│         │ Register│           │
│  └────┬────┘         └────┬────┘         └────┬────┘           │
│       │ ~1 cycle          │ ~1 cycle          │ ~1 cycle       │
│  ┌────▼────┐         ┌────▼────┐         ┌────▼────┐           │
│  │ L1 Cache│         │ L1 Cache│         │ L1 Cache│           │
│  │  32KB   │         │  32KB   │         │  32KB   │           │
│  └────┬────┘         └────┬────┘         └────┬────┘           │
│       │ ~4 cycles         │ ~4 cycles         │ ~4 cycles      │
│  ┌────▼────┐         ┌────▼────┐         ┌────▼────┐           │
│  │ L2 Cache│         │ L2 Cache│         │ L2 Cache│           │
│  │  256KB  │         │  256KB  │         │  256KB  │           │
│  └────┬────┘         └────┬────┘         └────┬────┘           │
│       │ ~12 cycles        │ ~12 cycles        │ ~12 cycles     │
│       └───────────────────┼───────────────────┘                │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │   L3 Cache  │ ~40 cycles                  │
│                    │    8-32MB   │ (shared)                    │
│                    └──────┬──────┘                              │
│                           │ ~100-300 cycles                     │
│                    ┌──────▼──────┐                              │
│                    │ Main Memory │                              │
│                    │    DDR4     │                              │
│                    └─────────────┘                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 캐시 라인 크기

| CPU | 캐시 라인 크기 |
|-----|---------------|
| Intel x86-64 | 64 bytes |
| AMD x86-64 | 64 bytes |
| ARM (대부분) | 64 bytes |
| Apple M1/M2 | 128 bytes |
| POWER | 128 bytes |

### MESI 프로토콜

캐시 일관성(Cache Coherence)을 위한 프로토콜입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESI 상태 전이                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  상태:                                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ M (Modified)  : 수정됨, 이 캐시만 유효                    │   │
│  │ E (Exclusive) : 단독 소유, 수정 안됨                      │   │
│  │ S (Shared)    : 여러 캐시에 존재, 읽기만 가능             │   │
│  │ I (Invalid)   : 무효화됨                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  False Sharing 시 상태 전이:                                     │
│                                                                 │
│    Core 0              Bus              Core 1                  │
│  ┌─────────┐                          ┌─────────┐              │
│  │ Line: M │ ─────Write A────────────►│ Line: I │              │
│  └─────────┘       Invalidate         └─────────┘              │
│                                                                 │
│  ┌─────────┐                          ┌─────────┐              │
│  │ Line: I │ ◄────Write B─────────────│ Line: M │              │
│  └─────────┘       Invalidate         └─────────┘              │
│                                                                 │
│  Core 0이 A를 쓰면 → Core 1의 캐시 라인 무효화                   │
│  Core 1이 B를 쓰면 → Core 0의 캐시 라인 무효화                   │
│  → 계속 서로 무효화하며 "핑퐁" 발생                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## False Sharing 발생 조건

### 1. 배열/구조체의 인접 요소

```c
// False Sharing 발생
int counters[NUM_THREADS];  // 4바이트 * N개가 연속 배치

void thread_func(int id) {
    for (int i = 0; i < ITERATIONS; i++) {
        counters[id]++;  // 다른 스레드의 카운터와 같은 캐시 라인
    }
}
```

### 2. 구조체 내 필드

```c
// False Sharing 발생
struct SharedData {
    volatile int counter_a;  // Thread 1이 사용
    volatile int counter_b;  // Thread 2가 사용
    volatile int counter_c;  // Thread 3이 사용
    volatile int counter_d;  // Thread 4가 사용
};  // 16 bytes - 모두 같은 캐시 라인에!
```

### 3. 전역 변수 배치

```c
// False Sharing 가능
int global_a;  // 컴파일러가 인접 배치할 수 있음
int global_b;
int global_c;
```

### 4. 동적 할당된 객체

```c
// malloc이 연속 주소 반환 시 False Sharing 가능
ThreadData* data[NUM_THREADS];
for (int i = 0; i < NUM_THREADS; i++) {
    data[i] = malloc(sizeof(ThreadData));  // 연속 주소 가능
}
```

---

## 성능 영향

### 벤치마크 예제

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>

#define NUM_THREADS 4
#define ITERATIONS 100000000

// False Sharing 발생
struct {
    volatile long counter[NUM_THREADS];
} shared_bad;

// False Sharing 방지 (패딩)
struct {
    volatile long counter;
    char padding[56];  // 64 - 8 = 56
} shared_good[NUM_THREADS];

void* increment_bad(void* arg) {
    int id = *(int*)arg;
    for (long i = 0; i < ITERATIONS; i++) {
        shared_bad.counter[id]++;
    }
    return NULL;
}

void* increment_good(void* arg) {
    int id = *(int*)arg;
    for (long i = 0; i < ITERATIONS; i++) {
        shared_good[id].counter++;
    }
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    int ids[NUM_THREADS];
    struct timespec start, end;

    // Bad (False Sharing)
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < NUM_THREADS; i++) {
        ids[i] = i;
        pthread_create(&threads[i], NULL, increment_bad, &ids[i]);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    double bad_time = (end.tv_sec - start.tv_sec) +
                      (end.tv_nsec - start.tv_nsec) / 1e9;

    // Good (No False Sharing)
    clock_gettime(CLOCK_MONOTONIC, &start);
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, increment_good, &ids[i]);
    }
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }
    clock_gettime(CLOCK_MONOTONIC, &end);
    double good_time = (end.tv_sec - start.tv_sec) +
                       (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("False Sharing:    %.3f seconds\n", bad_time);
    printf("No False Sharing: %.3f seconds\n", good_time);
    printf("Speedup: %.1fx\n", bad_time / good_time);

    return 0;
}
```

### 일반적인 결과

```
┌─────────────────────────────────────────────────────────────────┐
│                    벤치마크 결과 예시                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  설정: 4 코어, 각 스레드 1억 회 증가                             │
│                                                                 │
│  False Sharing:    12.5 seconds                                 │
│  No False Sharing:  0.8 seconds                                 │
│  Speedup: 15.6x                                                 │
│                                                                 │
│  코어 수 증가 시:                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Cores │ False Sharing │ No False Sharing │ Slowdown    │   │
│  │───────│───────────────│──────────────────│─────────────│   │
│  │   2   │     3.2s      │      0.4s        │    8x       │   │
│  │   4   │    12.5s      │      0.8s        │   16x       │   │
│  │   8   │    45.0s      │      1.6s        │   28x       │   │
│  │  16   │   180.0s      │      3.2s        │   56x       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  코어가 많을수록 False Sharing 영향이 기하급수적으로 증가!       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 탐지 방법

### 1. Linux perf

```bash
# 캐시 미스 확인
perf stat -e cache-misses,cache-references,L1-dcache-load-misses ./program

# 더 상세한 분석
perf record -e cache-misses ./program
perf report

# False Sharing 특화 이벤트 (Intel)
perf stat -e intel_pt//,mem_load_l3_miss_retired.remote_hitm ./program
```

### 2. Intel VTune

```bash
# Memory Access Analysis
vtune -collect memory-access ./program

# 결과 확인
vtune -report hotspots -r r000ma
```

### 3. Valgrind (cachegrind)

```bash
valgrind --tool=cachegrind ./program
cg_annotate cachegrind.out.<pid>
```

### 4. perf c2c (Cache-to-Cache)

```bash
# False Sharing 직접 탐지 (Linux 4.2+)
perf c2c record ./program
perf c2c report

# 결과 예시:
# =================================================
#            Shared Data Cache Line Distribution
# =================================================
# Rmt  LLC   Lcl   Total   Peer    Symbol
# Hit  Hit   Hit   Hits    Loads
# ---  ---   ---   -----   -----   ------
#  85%   5%  10%   10000   shared_bad.counter
#         ↑
#    Remote HITM이 높으면 False Sharing 의심!
```

### 5. 코드 검사

```c
// 컴파일 타임에 구조체 크기와 정렬 확인
#include <stddef.h>
#include <stdio.h>

struct BadStruct {
    int a;
    int b;
};

int main() {
    printf("Size: %zu, Alignment: %zu\n",
           sizeof(struct BadStruct),
           _Alignof(struct BadStruct));

    // 오프셋 확인
    printf("Offset of a: %zu\n", offsetof(struct BadStruct, a));
    printf("Offset of b: %zu\n", offsetof(struct BadStruct, b));

    // 같은 캐시 라인인지 확인
    #define CACHE_LINE_SIZE 64
    if (offsetof(struct BadStruct, a) / CACHE_LINE_SIZE ==
        offsetof(struct BadStruct, b) / CACHE_LINE_SIZE) {
        printf("WARNING: a and b are on the same cache line!\n");
    }
    return 0;
}
```

---

## 해결 방법

### 1. 패딩 (Padding)

```c
#define CACHE_LINE_SIZE 64

// 방법 1: 수동 패딩
struct PaddedCounter {
    volatile long value;
    char padding[CACHE_LINE_SIZE - sizeof(long)];
};

// 방법 2: 정렬 속성 사용 (GCC/Clang)
struct AlignedCounter {
    volatile long value;
} __attribute__((aligned(CACHE_LINE_SIZE)));

// 방법 3: C11 alignas
#include <stdalign.h>
struct AlignedCounter {
    alignas(CACHE_LINE_SIZE) volatile long value;
};

// 방법 4: C++11 alignas
struct alignas(64) AlignedCounter {
    volatile long value;
};
```

### 2. 스레드 로컬 저장소 (TLS)

```c
// 각 스레드가 자신의 카운터를 가짐
__thread long local_counter = 0;

void thread_func() {
    for (int i = 0; i < ITERATIONS; i++) {
        local_counter++;  // False Sharing 없음!
    }
}

// 나중에 합산
long get_total() {
    // 모든 스레드의 local_counter 합산 필요
}
```

### 3. 배열 분리

```c
// Bad: 연속 배열
int counters[NUM_THREADS];

// Good: 각 카운터를 별도 캐시 라인에
struct {
    int value;
    char padding[60];
} counters[NUM_THREADS];

// 또는 동적 할당으로 분리
int* counters[NUM_THREADS];
for (int i = 0; i < NUM_THREADS; i++) {
    counters[i] = aligned_alloc(64, 64);  // 캐시 라인 정렬
}
```

### 4. 데이터 재구성

```c
// Bad: AoS (Array of Structures) - False Sharing 발생
struct Entity {
    float x, y, z;    // Thread 1이 position 업데이트
    float health;     // Thread 2가 health 업데이트
};
Entity entities[1000];

// Good: SoA (Structure of Arrays) - False Sharing 감소
struct Positions {
    float x[1000];
    float y[1000];
    float z[1000];
};
struct Healths {
    float health[1000];
};
```

### 5. 언어별 지원

#### C++17 hardware_destructive_interference_size

```cpp
#include <new>

// 캐시 라인 크기를 컴파일 타임에 얻기
constexpr size_t cache_line = std::hardware_destructive_interference_size;

struct alignas(cache_line) CacheAlignedCounter {
    std::atomic<long> value{0};
};

// 또는 패딩 계산에 사용
struct PaddedCounter {
    std::atomic<long> value{0};
    char padding[cache_line - sizeof(std::atomic<long>)];
};
```

#### Java @Contended

```java
// JDK 8+, -XX:-RestrictContended 필요
import sun.misc.Contended;

public class Counter {
    @Contended
    volatile long counter1;

    @Contended
    volatile long counter2;
}
```

#### Rust (crossbeam)

```rust
use crossbeam_utils::CachePadded;

struct Counters {
    counter1: CachePadded<AtomicUsize>,
    counter2: CachePadded<AtomicUsize>,
}
```

---

## 실전 예제

### Thread Pool의 작업 큐

```c
#include <stdatomic.h>

#define CACHE_LINE_SIZE 64
#define NUM_WORKERS 8

// Bad: 모든 워커의 큐 인덱스가 인접
struct BadWorkerQueues {
    atomic_int head[NUM_WORKERS];
    atomic_int tail[NUM_WORKERS];
};

// Good: 각 워커의 데이터를 캐시 라인 단위로 분리
struct WorkerQueue {
    atomic_int head;
    atomic_int tail;
    char padding[CACHE_LINE_SIZE - 2 * sizeof(atomic_int)];
} __attribute__((aligned(CACHE_LINE_SIZE)));

struct GoodWorkerQueues {
    struct WorkerQueue queues[NUM_WORKERS];
};
```

### 통계 수집기

```c
#include <stdatomic.h>

#define CACHE_LINE_SIZE 64
#define NUM_THREADS 8

// 스레드별 통계 (False Sharing 방지)
struct ThreadStats {
    atomic_long requests_processed;
    atomic_long bytes_transferred;
    atomic_long errors;
    char padding[CACHE_LINE_SIZE - 3 * sizeof(atomic_long)];
} __attribute__((aligned(CACHE_LINE_SIZE)));

ThreadStats stats[NUM_THREADS];

// 스레드 함수
void worker_thread(int id) {
    while (running) {
        // 작업 처리...
        atomic_fetch_add(&stats[id].requests_processed, 1);
        atomic_fetch_add(&stats[id].bytes_transferred, bytes);
    }
}

// 전체 통계 집계 (가끔 호출)
void get_total_stats(long* requests, long* bytes, long* errors) {
    *requests = 0;
    *bytes = 0;
    *errors = 0;
    for (int i = 0; i < NUM_THREADS; i++) {
        *requests += atomic_load(&stats[i].requests_processed);
        *bytes += atomic_load(&stats[i].bytes_transferred);
        *errors += atomic_load(&stats[i].errors);
    }
}
```

### Lock-Free 카운터 (분산 카운팅)

```cpp
#include <atomic>
#include <new>
#include <thread>

class DistributedCounter {
private:
    static constexpr size_t CacheLineSize =
        std::hardware_destructive_interference_size;

    struct alignas(CacheLineSize) PaddedCounter {
        std::atomic<long> value{0};
    };

    PaddedCounter* counters;
    size_t num_slots;

public:
    DistributedCounter(size_t slots = std::thread::hardware_concurrency())
        : num_slots(slots) {
        counters = new PaddedCounter[num_slots];
    }

    ~DistributedCounter() {
        delete[] counters;
    }

    void increment() {
        // 스레드 ID를 슬롯으로 매핑 (간단한 해싱)
        size_t slot = std::hash<std::thread::id>{}(
            std::this_thread::get_id()) % num_slots;
        counters[slot].value.fetch_add(1, std::memory_order_relaxed);
    }

    long get() const {
        long total = 0;
        for (size_t i = 0; i < num_slots; i++) {
            total += counters[i].value.load(std::memory_order_relaxed);
        }
        return total;
    }
};
```

### Spinlock with Backoff (캐시 친화적)

```c
#include <stdatomic.h>
#include <sched.h>

// 캐시 라인 정렬된 스핀락
struct CacheAlignedSpinlock {
    atomic_int locked;
    char padding[60];
} __attribute__((aligned(64)));

void spin_lock(struct CacheAlignedSpinlock* lock) {
    int backoff = 1;

    while (true) {
        // 읽기만 (캐시 라인 무효화 없음)
        while (atomic_load_explicit(&lock->locked, memory_order_relaxed)) {
            for (int i = 0; i < backoff; i++) {
                __builtin_ia32_pause();  // CPU 힌트
            }
            if (backoff < 1024) backoff *= 2;
        }

        // CAS 시도 (여기서만 쓰기 발생)
        int expected = 0;
        if (atomic_compare_exchange_weak_explicit(
                &lock->locked, &expected, 1,
                memory_order_acquire, memory_order_relaxed)) {
            return;
        }
        backoff = 1;
    }
}

void spin_unlock(struct CacheAlignedSpinlock* lock) {
    atomic_store_explicit(&lock->locked, 0, memory_order_release);
}
```

---

## 요약

### False Sharing 핵심

| 항목 | 설명 |
|------|------|
| **원인** | 독립적 데이터가 같은 캐시 라인에 위치 |
| **증상** | 캐시 무효화 폭증, 성능 급격히 저하 |
| **영향** | 코어 수 증가 시 더 심해짐 |
| **해결** | 패딩, 정렬, TLS, 데이터 재구성 |

### 체크리스트

```
┌─────────────────────────────────────────────────────────────────┐
│               False Sharing 방지 체크리스트                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  설계 시:                                                        │
│  □ 스레드별로 자주 쓰는 데이터 식별                              │
│  □ 해당 데이터를 별도 캐시 라인에 배치 계획                       │
│  □ 배열 대신 TLS 고려                                           │
│                                                                 │
│  구현 시:                                                        │
│  □ 구조체에 패딩 추가 또는 정렬 속성 사용                         │
│  □ 캐시 라인 크기(64B)를 상수로 정의                             │
│  □ 동적 할당 시 aligned_alloc 사용                               │
│                                                                 │
│  검증 시:                                                        │
│  □ perf c2c로 False Sharing 탐지                                │
│  □ 코어 수 늘려서 스케일링 테스트                                │
│  □ 캐시 미스 비율 확인                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 성능 최적화 우선순위

1. **측정 먼저**: 실제로 False Sharing이 문제인지 확인
2. **핫 패스 집중**: 자주 실행되는 코드만 최적화
3. **메모리 vs 성능**: 패딩은 메모리 사용량 증가
4. **플랫폼 고려**: 캐시 라인 크기가 다를 수 있음

---

## 관련 문서

- [Thread Local Storage](../01-fundamentals/05-thread-local-storage.md) - TLS로 False Sharing 방지
- [성능 튜닝](./performance-tuning.md) - 전반적인 성능 최적화
- [Memory Ordering](../05-lock-free-programming/02-memory-ordering.md) - 캐시와 메모리 모델
- [Atomic Operations](../02-synchronization/04-atomic-operations.md) - 원자적 연산

---

## 참고 자료

- [What Every Programmer Should Know About Memory - Ulrich Drepper](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- [Avoiding and Identifying False Sharing - Intel](https://www.intel.com/content/www/us/en/developer/articles/technical/avoiding-and-identifying-false-sharing-among-threads.html)
- [perf c2c Documentation](https://man7.org/linux/man-pages/man1/perf-c2c.1.html)
- [std::hardware_destructive_interference_size - cppreference](https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size)
