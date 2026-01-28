# Seqlock (Sequence Lock)

## 1. Seqlock 개요

### 1.1 Seqlock이란?

Seqlock은 **Writer 우선**의 경량 동기화 메커니즘으로, Reader가 Writer를 블록하지 않고 Writer가 즉시 쓰기를 수행할 수 있습니다. Reader는 읽기 중 쓰기가 발생했는지 감지하고, 발생했다면 재시도합니다.

```
Seqlock 동작 원리:

시퀀스 카운터: 0 (짝수 = 안전)
                ↓
Writer 진입:   1 (홀수 = 쓰기 중)
                ↓
Writer 완료:   2 (짝수 = 안전)

[Reader 관점]
1. 시작 시 시퀀스 읽기 (seq_before)
2. 데이터 읽기
3. 끝에서 시퀀스 읽기 (seq_after)
4. seq_before != seq_after 또는 홀수면 → 재시도
```

### 1.2 RCU와의 비교

```
┌────────────────┬──────────────────┬──────────────────┐
│     특성       │      RCU         │     Seqlock      │
├────────────────┼──────────────────┼──────────────────┤
│ Reader 블록    │       ✗          │        ✗         │
│ Writer 블록    │   Grace Period   │        ✗         │
│ Reader 재시도  │       ✗          │        ✓         │
│ 메모리 복사    │       ✓          │        ✗         │
│ 메모리 오버헤드│      높음        │       낮음       │
│ 적합한 경우    │ 포인터 기반 구조 │  단순 값 데이터  │
└────────────────┴──────────────────┴──────────────────┘
```

### 1.3 핵심 특성

1. **Writer 우선**: Writer가 Reader를 기다리지 않음
2. **락-프리 읽기**: Reader는 락 획득 없이 읽기
3. **낙관적 동시성**: Reader는 실패 시 재시도
4. **경량**: 메모리 할당/해제 불필요

## 2. Seqlock 구현

### 2.1 기본 구현

```c
#include <stdatomic.h>
#include <stdbool.h>

typedef struct {
    atomic_uint sequence;    // 시퀀스 카운터
    pthread_spinlock_t lock; // Writer 간 상호배제
} seqlock_t;

// 초기화
void seqlock_init(seqlock_t *sl)
{
    atomic_init(&sl->sequence, 0);
    pthread_spin_init(&sl->lock, PTHREAD_PROCESS_PRIVATE);
}

// Writer: 쓰기 시작
void write_seqlock(seqlock_t *sl)
{
    pthread_spin_lock(&sl->lock);
    // 시퀀스를 홀수로 (쓰기 중 표시)
    atomic_fetch_add_explicit(&sl->sequence, 1, memory_order_release);
    atomic_thread_fence(memory_order_seq_cst);
}

// Writer: 쓰기 완료
void write_sequnlock(seqlock_t *sl)
{
    atomic_thread_fence(memory_order_seq_cst);
    // 시퀀스를 짝수로 (쓰기 완료)
    atomic_fetch_add_explicit(&sl->sequence, 1, memory_order_release);
    pthread_spin_unlock(&sl->lock);
}

// Reader: 시퀀스 읽기 (시작)
unsigned read_seqbegin(seqlock_t *sl)
{
    unsigned seq;
    do {
        seq = atomic_load_explicit(&sl->sequence, memory_order_acquire);
    } while (seq & 1);  // 홀수면 (쓰기 중) 대기

    return seq;
}

// Reader: 읽기 유효성 검증
bool read_seqretry(seqlock_t *sl, unsigned start_seq)
{
    atomic_thread_fence(memory_order_acquire);
    unsigned end_seq = atomic_load_explicit(&sl->sequence, memory_order_acquire);
    return start_seq != end_seq;  // true면 재시도 필요
}
```

### 2.2 사용 예시

```c
// 보호할 데이터
struct timestamp {
    uint32_t seconds;
    uint32_t nanoseconds;
};

seqlock_t time_lock;
struct timestamp current_time;

// Writer: 시간 업데이트
void update_time(uint32_t secs, uint32_t nsecs)
{
    write_seqlock(&time_lock);

    current_time.seconds = secs;
    current_time.nanoseconds = nsecs;

    write_sequnlock(&time_lock);
}

// Reader: 시간 읽기
struct timestamp read_time(void)
{
    struct timestamp ts;
    unsigned seq;

    do {
        seq = read_seqbegin(&time_lock);

        // 데이터 복사
        ts.seconds = current_time.seconds;
        ts.nanoseconds = current_time.nanoseconds;

    } while (read_seqretry(&time_lock, seq));

    return ts;
}
```

### 2.3 Linux 커널 스타일 구현

```c
// Linux 커널의 seqlock
typedef struct {
    unsigned sequence;
    spinlock_t lock;
} seqlock_t;

#define SEQLOCK_UNLOCKED { 0, SPIN_LOCK_UNLOCKED }

// seqcount만 사용 (Writer 락 없음)
typedef struct {
    unsigned sequence;
} seqcount_t;

// Read Section
static inline unsigned read_seqcount_begin(const seqcount_t *s)
{
    unsigned ret;

repeat:
    ret = READ_ONCE(s->sequence);
    if (unlikely(ret & 1)) {
        cpu_relax();
        goto repeat;
    }
    smp_rmb();  // Read Memory Barrier
    return ret;
}

static inline int read_seqcount_retry(const seqcount_t *s, unsigned start)
{
    smp_rmb();
    return unlikely(s->sequence != start);
}

// Write Section
static inline void write_seqcount_begin(seqcount_t *s)
{
    s->sequence++;
    smp_wmb();  // Write Memory Barrier
}

static inline void write_seqcount_end(seqcount_t *s)
{
    smp_wmb();
    s->sequence++;
}

// Full seqlock with spinlock
static inline void write_seqlock(seqlock_t *sl)
{
    spin_lock(&sl->lock);
    write_seqcount_begin(&sl->seqcount);
}

static inline void write_sequnlock(seqlock_t *sl)
{
    write_seqcount_end(&sl->seqcount);
    spin_unlock(&sl->lock);
}
```

## 3. 고급 Seqlock 패턴

### 3.1 Raw Seqcount (Lock 없는 변형)

여러 Writer가 없거나 외부에서 동기화할 때 사용합니다.

```c
// Writer 락 없이 seqcount만 사용
typedef struct {
    atomic_uint seq;
} raw_seqcount_t;

// 단일 Writer 상황
void single_writer_update(raw_seqcount_t *sc, data_t *data, data_t new_val)
{
    // Writer가 하나뿐이므로 락 불필요
    atomic_fetch_add_explicit(&sc->seq, 1, memory_order_release);
    atomic_thread_fence(memory_order_seq_cst);

    *data = new_val;

    atomic_thread_fence(memory_order_seq_cst);
    atomic_fetch_add_explicit(&sc->seq, 1, memory_order_release);
}

// 외부 락으로 보호되는 상황
pthread_mutex_t external_lock;

void externally_synchronized_update(raw_seqcount_t *sc, data_t *data, data_t new_val)
{
    pthread_mutex_lock(&external_lock);

    // seqcount만 업데이트
    unsigned seq = atomic_load(&sc->seq);
    atomic_store_explicit(&sc->seq, seq + 1, memory_order_release);
    atomic_thread_fence(memory_order_seq_cst);

    *data = new_val;

    atomic_thread_fence(memory_order_seq_cst);
    atomic_store_explicit(&sc->seq, seq + 2, memory_order_release);

    pthread_mutex_unlock(&external_lock);
}
```

### 3.2 Seqlock + RCU 조합

대규모 데이터에 Seqlock과 RCU를 조합합니다.

```c
// 큰 구조체: RCU로 포인터 교체
// 작은 메타데이터: Seqlock으로 보호
struct large_data {
    // 큰 데이터...
    char buffer[4096];
    struct rcu_head rcu;
};

struct metadata {
    uint64_t version;
    uint64_t checksum;
    struct large_data __rcu *data;
};

seqcount_t meta_seq;
struct metadata global_meta;

// Reader: 메타데이터 + 데이터 일관성 있게 읽기
int read_consistent(char *buf, size_t len)
{
    struct large_data *data;
    uint64_t version, checksum;
    unsigned seq;

    rcu_read_lock();
    do {
        seq = read_seqcount_begin(&meta_seq);

        version = global_meta.version;
        checksum = global_meta.checksum;
        data = rcu_dereference(global_meta.data);

        // 데이터 복사
        if (data) {
            memcpy(buf, data->buffer, min(len, sizeof(data->buffer)));
        }

    } while (read_seqcount_retry(&meta_seq, seq));

    rcu_read_unlock();

    return verify_checksum(buf, len, checksum) ? 0 : -1;
}

// Writer: 메타데이터 + 데이터 업데이트
void update_data(const char *new_data, size_t len)
{
    struct large_data *new = kmalloc(sizeof(*new), GFP_KERNEL);
    struct large_data *old;

    memcpy(new->buffer, new_data, len);

    write_seqcount_begin(&meta_seq);

    global_meta.version++;
    global_meta.checksum = compute_checksum(new_data, len);
    old = rcu_replace_pointer(global_meta.data, new, true);

    write_seqcount_end(&meta_seq);

    if (old) {
        call_rcu(&old->rcu, free_large_data);
    }
}
```

### 3.3 Latch (Read-Side 스케일링)

```c
// 두 개의 버퍼를 번갈아 사용
struct seqlatch {
    atomic_uint sequence;
    data_t data[2];  // Double buffering
};

// Writer
void latch_write(struct seqlatch *latch, data_t new_val)
{
    unsigned seq = atomic_load(&latch->sequence);
    unsigned idx = (seq + 1) & 1;  // 다음 버퍼

    // 비활성 버퍼에 쓰기
    latch->data[idx] = new_val;

    atomic_thread_fence(memory_order_release);

    // 시퀀스 증가로 활성 버퍼 전환
    atomic_store_explicit(&latch->sequence, seq + 1, memory_order_release);
}

// Reader
data_t latch_read(struct seqlatch *latch)
{
    unsigned seq = atomic_load_explicit(&latch->sequence, memory_order_acquire);
    unsigned idx = seq & 1;  // 현재 활성 버퍼

    atomic_thread_fence(memory_order_acquire);

    return latch->data[idx];  // 항상 일관된 데이터
}
```

## 4. 실전 사용 사례

### 4.1 시스템 시간 (jiffies/xtime)

Linux 커널에서 Seqlock의 대표적 사용 사례입니다.

```c
// Linux 커널 xtime 구조체
struct timekeeper {
    struct tk_read_base tkr_mono;
    struct tk_read_base tkr_raw;
    u64 xtime_sec;
    unsigned long ktime_sec;
    struct timespec64 wall_to_monotonic;
    // ... 더 많은 필드
};

static struct {
    seqcount_t seq;
    struct timekeeper timekeeper;
} tk_core;

// 시간 읽기 (매우 빈번히 호출됨)
void ktime_get_ts64(struct timespec64 *ts)
{
    struct timekeeper *tk = &tk_core.timekeeper;
    unsigned int seq;

    do {
        seq = read_seqcount_begin(&tk_core.seq);
        ts->tv_sec = tk->xtime_sec;
        ts->tv_nsec = timekeeping_get_ns(&tk->tkr_mono);
    } while (read_seqcount_retry(&tk_core.seq, seq));
}

// 시간 업데이트 (Timer Interrupt에서 호출)
void timekeeping_update(void)
{
    struct timekeeper *tk = &tk_core.timekeeper;

    write_seqcount_begin(&tk_core.seq);

    // 시간 업데이트 로직...
    tk->xtime_sec++;

    write_seqcount_end(&tk_core.seq);
}
```

### 4.2 통계 카운터

```c
// 여러 관련 통계를 원자적으로 읽기
struct network_stats {
    uint64_t rx_packets;
    uint64_t rx_bytes;
    uint64_t tx_packets;
    uint64_t tx_bytes;
    uint64_t errors;
};

struct interface {
    seqcount_t stats_seq;
    struct network_stats stats;
    pthread_spinlock_t stats_lock;  // Writer 간 동기화
};

// 패킷 수신 (빈번)
void on_packet_received(struct interface *iface, size_t bytes)
{
    pthread_spin_lock(&iface->stats_lock);
    write_seqcount_begin(&iface->stats_seq);

    iface->stats.rx_packets++;
    iface->stats.rx_bytes += bytes;

    write_seqcount_end(&iface->stats_seq);
    pthread_spin_unlock(&iface->stats_lock);
}

// 통계 조회
struct network_stats get_stats(struct interface *iface)
{
    struct network_stats stats;
    unsigned seq;

    do {
        seq = read_seqcount_begin(&iface->stats_seq);

        stats = iface->stats;  // 구조체 복사

    } while (read_seqcount_retry(&iface->stats_seq, seq));

    return stats;
}

// 통계 초기화
void reset_stats(struct interface *iface)
{
    pthread_spin_lock(&iface->stats_lock);
    write_seqcount_begin(&iface->stats_seq);

    memset(&iface->stats, 0, sizeof(iface->stats));

    write_seqcount_end(&iface->stats_seq);
    pthread_spin_unlock(&iface->stats_lock);
}
```

### 4.3 경로 조회 (dcache)

```c
// Linux VFS dcache rename seqlock
seqlock_t rename_lock;

// 경로 조회 (락-프리)
struct dentry *d_lookup(const struct dentry *parent, const struct qstr *name)
{
    struct dentry *dentry;
    unsigned seq;

retry:
    seq = read_seqbegin(&rename_lock);

    dentry = __d_lookup(parent, name);

    if (read_seqretry(&rename_lock, seq)) {
        goto retry;
    }

    return dentry;
}

// 디렉토리 이름 변경 (드묾)
int rename_directory(struct dentry *old, struct dentry *new)
{
    write_seqlock(&rename_lock);

    // rename 로직...
    swap_dentries(old, new);

    write_sequnlock(&rename_lock);

    return 0;
}
```

### 4.4 좌표/위치 정보

```c
// 3D 좌표 (12바이트 - 원자적 연산 불가)
struct position {
    float x, y, z;
};

struct game_object {
    seqcount_t pos_seq;
    struct position pos;
    // Writer는 하나 (해당 객체 소유 스레드)
};

// 위치 업데이트 (소유 스레드만)
void move_object(struct game_object *obj, float dx, float dy, float dz)
{
    write_seqcount_begin(&obj->pos_seq);

    obj->pos.x += dx;
    obj->pos.y += dy;
    obj->pos.z += dz;

    write_seqcount_end(&obj->pos_seq);
}

// 다른 스레드에서 위치 읽기
struct position get_position(struct game_object *obj)
{
    struct position pos;
    unsigned seq;

    do {
        seq = read_seqcount_begin(&obj->pos_seq);

        pos = obj->pos;

    } while (read_seqcount_retry(&obj->pos_seq, seq));

    return pos;
}

// 충돌 검사 등
float distance_between(struct game_object *a, struct game_object *b)
{
    struct position pa, pb;
    unsigned seq_a, seq_b;

    // 두 객체의 위치를 일관성 있게 읽기
    do {
        seq_a = read_seqcount_begin(&a->pos_seq);
        seq_b = read_seqcount_begin(&b->pos_seq);

        pa = a->pos;
        pb = b->pos;

    } while (read_seqcount_retry(&a->pos_seq, seq_a) ||
             read_seqcount_retry(&b->pos_seq, seq_b));

    float dx = pa.x - pb.x;
    float dy = pa.y - pb.y;
    float dz = pa.z - pb.z;

    return sqrtf(dx*dx + dy*dy + dz*dz);
}
```

## 5. C++ Seqlock 구현

### 5.1 현대 C++ 구현

```cpp
#include <atomic>
#include <thread>

template<typename T>
class Seqlock {
private:
    std::atomic<unsigned> sequence_{0};
    T data_;
    alignas(64) char padding_[64];  // False sharing 방지

public:
    // Writer
    template<typename F>
    void write(F&& func) {
        auto seq = sequence_.load(std::memory_order_relaxed);
        sequence_.store(seq + 1, std::memory_order_release);
        std::atomic_thread_fence(std::memory_order_seq_cst);

        func(data_);

        std::atomic_thread_fence(std::memory_order_seq_cst);
        sequence_.store(seq + 2, std::memory_order_release);
    }

    // Reader
    T read() const {
        T result;
        unsigned seq0, seq1;

        do {
            seq0 = sequence_.load(std::memory_order_acquire);
            while (seq0 & 1) {  // 쓰기 중이면 대기
                std::this_thread::yield();
                seq0 = sequence_.load(std::memory_order_acquire);
            }

            std::atomic_thread_fence(std::memory_order_acquire);
            result = data_;
            std::atomic_thread_fence(std::memory_order_acquire);

            seq1 = sequence_.load(std::memory_order_acquire);
        } while (seq0 != seq1);

        return result;
    }
};

// 사용 예시
struct Timestamp {
    uint64_t seconds;
    uint64_t nanoseconds;
};

Seqlock<Timestamp> time_lock;

// Writer
void update_time() {
    time_lock.write([](Timestamp& ts) {
        ts.seconds = get_current_seconds();
        ts.nanoseconds = get_current_nanoseconds();
    });
}

// Reader
Timestamp read_time() {
    return time_lock.read();
}
```

### 5.2 Thread-Safe Seqlock (Multiple Writers)

```cpp
#include <atomic>
#include <mutex>

template<typename T>
class ThreadSafeSeqlock {
private:
    mutable std::atomic<unsigned> sequence_{0};
    T data_;
    std::mutex writer_mutex_;

public:
    // 여러 Writer 지원
    template<typename F>
    void write(F&& func) {
        std::lock_guard<std::mutex> lock(writer_mutex_);

        sequence_.fetch_add(1, std::memory_order_release);
        std::atomic_thread_fence(std::memory_order_seq_cst);

        func(data_);

        std::atomic_thread_fence(std::memory_order_seq_cst);
        sequence_.fetch_add(1, std::memory_order_release);
    }

    // Reader (변경 없음)
    T read() const {
        T result;
        unsigned seq0, seq1;

        do {
            seq0 = sequence_.load(std::memory_order_acquire);
            while (seq0 & 1) {
                std::this_thread::yield();
                seq0 = sequence_.load(std::memory_order_acquire);
            }

            std::atomic_thread_fence(std::memory_order_acquire);
            result = data_;
            std::atomic_thread_fence(std::memory_order_acquire);

            seq1 = sequence_.load(std::memory_order_acquire);
        } while (seq0 != seq1);

        return result;
    }

    // Optimistic read with callback
    template<typename F>
    auto read_with(F&& func) const {
        unsigned seq0, seq1;
        decltype(func(std::declval<const T&>())) result;

        do {
            seq0 = sequence_.load(std::memory_order_acquire);
            while (seq0 & 1) {
                std::this_thread::yield();
                seq0 = sequence_.load(std::memory_order_acquire);
            }

            std::atomic_thread_fence(std::memory_order_acquire);
            result = func(data_);
            std::atomic_thread_fence(std::memory_order_acquire);

            seq1 = sequence_.load(std::memory_order_acquire);
        } while (seq0 != seq1);

        return result;
    }
};
```

### 5.3 Folly 스타일 Seqlock

```cpp
// Facebook Folly 라이브러리 영감
template<typename T>
class SeqlockValue {
    static_assert(std::is_trivially_copyable_v<T>,
                  "T must be trivially copyable");

private:
    struct alignas(64) Storage {
        std::atomic<uint32_t> seq{0};
        T value{};
    };

    Storage storage_;

public:
    void store(const T& value) {
        auto& s = storage_;
        auto seq = s.seq.load(std::memory_order_relaxed);

        s.seq.store(seq + 1, std::memory_order_release);
        std::atomic_signal_fence(std::memory_order_acq_rel);

        s.value = value;

        std::atomic_signal_fence(std::memory_order_acq_rel);
        s.seq.store(seq + 2, std::memory_order_release);
    }

    T load() const {
        auto& s = storage_;
        T result;
        uint32_t seq0, seq1;

        do {
            seq0 = s.seq.load(std::memory_order_acquire);

            // Spin while write in progress
            while (seq0 & 1) {
                // Exponential backoff
                for (int i = 0; i < 4; ++i) {
                    __builtin_ia32_pause();
                }
                seq0 = s.seq.load(std::memory_order_acquire);
            }

            std::atomic_signal_fence(std::memory_order_acq_rel);
            result = s.value;
            std::atomic_signal_fence(std::memory_order_acq_rel);

            seq1 = s.seq.load(std::memory_order_acquire);
        } while (seq0 != seq1);

        return result;
    }
};
```

## 6. 성능 분석

### 6.1 Reader 오버헤드

```
Reader 오버헤드 비교 (나노초):

│  Seqlock (no contention)
│  ██  ~5-10ns
│
│  RCU
│  ██  ~5-10ns
│
│  RW Lock (read)
│  ████████████████  ~50-80ns
│
│  Mutex
│  ████████████████████████  ~80-120ns
│
└────────────────────────────────────────────
```

### 6.2 Writer 오버헤드

```
Writer 오버헤드 비교 (나노초):

│  Seqlock
│  ████  ~20-30ns
│
│  Mutex
│  ████████████  ~80-100ns
│
│  RW Lock (write)
│  ██████████████████  ~100-150ns
│
│  RCU (synchronize_rcu)
│  ████████████████████████████████████  ~50,000-100,000ns
│
└────────────────────────────────────────────
```

### 6.3 확장성 비교

```
처리량 vs CPU 수 (읽기 위주 워크로드):

Throughput
│
│                     Seqlock
│                   ╱
│                 ╱     RCU
│               ╱     ╱
│             ╱     ╱
│           ╱     ╱     RW Lock
│         ╱     ╱     ╱
│       ╱     ╱     ╱
│     ╱     ╱     ╱        Mutex
│   ╱     ╱     ╱      ────────────
│ ╱     ╱     ╱
└──────────────────────────────────→ CPUs
 1   4   8   16   32   64
```

### 6.4 재시도 확률

```
Writer 빈도에 따른 Reader 재시도 확률:

재시도 확률 (%)
│
│ 50% ─────────────────────────────╱
│                               ╱
│ 25% ───────────────────────╱
│                         ╱
│ 10% ─────────────────╱
│                   ╱
│  5% ───────────╱
│            ╱
│  1% ────╱
│     ╱
└───────────────────────────────────→ Writer 빈도
   1/s   10/s   100/s   1K/s   10K/s

주의: Writer가 매우 빈번하면 Reader 기아 가능
```

## 7. Seqlock 주의사항

### 7.1 일반적인 실수

```c
// 실수 1: 포인터 역참조
void bug1(void)
{
    struct node *ptr;
    unsigned seq;

    do {
        seq = read_seqbegin(&lock);
        ptr = global_ptr;  // 포인터 복사
    } while (read_seqretry(&lock, seq));

    // BUG! ptr이 이미 해제되었을 수 있음
    use(ptr->data);  // Use-After-Free 가능
}

// 해결: RCU와 조합하거나 데이터 자체를 복사
void correct1_rcu(void)
{
    struct node *ptr;
    unsigned seq;

    rcu_read_lock();  // RCU로 존재 보장
    do {
        seq = read_seqbegin(&lock);
        ptr = rcu_dereference(global_ptr);
    } while (read_seqretry(&lock, seq));

    use(ptr->data);  // 안전
    rcu_read_unlock();
}

void correct1_copy(void)
{
    data_t data;
    unsigned seq;

    do {
        seq = read_seqbegin(&lock);
        data = global_data;  // 값 복사
    } while (read_seqretry(&lock, seq));

    use(data);  // 안전
}
```

### 7.2 부적합한 사용 사례

```c
// 1. Writer가 매우 빈번한 경우
// Reader 기아 발생 가능
void bad_frequent_writer(void)
{
    while (true) {
        write_seqlock(&lock);
        update_data();  // 매우 빈번
        write_sequnlock(&lock);
    }
}

// 2. 쓰기 작업이 오래 걸리는 경우
void bad_long_write(void)
{
    write_seqlock(&lock);
    slow_operation();  // Reader가 계속 실패
    write_sequnlock(&lock);
}

// 3. 포인터 기반 데이터 구조
void bad_pointer_based(void)
{
    unsigned seq;
    do {
        seq = read_seqbegin(&lock);
        // 포인터 따라가기 - 중간에 해제될 수 있음
        node = head->next->next;  // DANGEROUS!
    } while (read_seqretry(&lock, seq));
}
```

### 7.3 Seqlock vs 다른 동기화

```
┌─────────────────────────────────────────────────────────────────┐
│                     동기화 방식 선택 가이드                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  데이터가 단순 값인가? ─── YES ──→ 쓰기 빈도 낮은가?            │
│         │                              │                        │
│        NO                         YES ─┼─ NO                    │
│         │                              │     │                  │
│         ▼                              ▼     ▼                  │
│  포인터 기반 구조                 Seqlock   Atomic/Lock         │
│         │                                                       │
│         ▼                                                       │
│  읽기가 압도적으로 많은가? ─── YES ──→ RCU                      │
│         │                                                       │
│        NO                                                       │
│         │                                                       │
│         ▼                                                       │
│  RW Lock 또는 Mutex                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 8. 요약

### Seqlock 적합성 체크리스트

| 조건 | 적합 여부 |
|------|-----------|
| 단순 값 데이터 (포인터 아님) | ✓ 적합 |
| 읽기가 쓰기보다 훨씬 많음 | ✓ 적합 |
| 쓰기 작업이 짧음 | ✓ 적합 |
| Writer 우선 정책 필요 | ✓ 적합 |
| 포인터 기반 구조 | ✗ RCU 사용 |
| 쓰기가 빈번함 | ✗ Lock 사용 |
| 긴 쓰기 작업 | ✗ Lock 사용 |

### 핵심 포인트

1. **장점**
   - Writer가 블록되지 않음
   - Reader 오버헤드 매우 낮음
   - 메모리 오버헤드 없음
   - 구현이 간단함

2. **단점**
   - Reader가 재시도해야 할 수 있음
   - 포인터 기반 데이터에 부적합
   - Writer 빈번 시 Reader 기아

3. **최적 사용 사례**
   - 시스템 시간 (jiffies)
   - 통계 카운터
   - 좌표/위치 데이터
   - 설정 값 읽기

Seqlock은 RCU와 함께 Linux 커널의 핵심 동기화 메커니즘으로, 특히 시간 관련 코드에서 광범위하게 사용됩니다. 단순 값 데이터의 빠른 읽기가 필요한 상황에서 탁월한 성능을 제공합니다.
