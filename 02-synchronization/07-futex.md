# Futex (Fast Userspace Mutex)

## 📌 개요

**Futex (Fast Userspace Mutex)**는 Linux 커널의 저수준 동기화 프리미티브로, **유저 스페이스와 커널 스페이스를 결합한 하이브리드 동기화 메커니즘**입니다. pthread mutex, semaphore, condition variable 등 대부분의 고수준 동기화 도구의 기반이 됩니다.

**핵심 아이디어**: 경합이 없을 때는 유저 스페이스에서만 동작(빠름), 경합이 있을 때만 커널 호출(느림)

---

## 🎯 등장 배경

### 전통적인 동기화 메커니즘의 문제

**System V Semaphore, POSIX Semaphore (sem_wait/sem_post)**:
```c
// 매번 커널 호출 필요
sem_wait(&sem);    // 시스템 콜 (항상 느림)
// Critical Section
sem_post(&sem);    // 시스템 콜 (항상 느림)
```

**문제점**:
- ⚠️ 경합이 없어도 항상 커널 모드 진입
- ⚠️ Context Switching 오버헤드
- ⚠️ 시스템 콜 비용: 수백 cycles

### Futex의 해결책

```
경합 없음 (Fast Path):
    User Space에서만 Atomic 연산 → 매우 빠름 (10-20 cycles)

경합 있음 (Slow Path):
    Kernel Space 진입 → 스레드 대기/깨우기
```

**성능 비교**:
| 상황 | 전통적 Semaphore | Futex |
|------|-----------------|-------|
| **경합 없음** | ~200 cycles (시스템 콜) | ~10 cycles (atomic) |
| **경합 있음** | ~200 cycles | ~200 cycles |

→ **경합이 드문 경우 20배 빠름!**

---

## 🏗️ Futex 아키텍처

### 기본 구조

```
User Space:
┌──────────────────────────┐
│  int futex_word = 0;     │ ← Atomic 변수 (공유 메모리)
│                          │
│  if (atomic_op(&futex))  │ ← Fast Path (경합 없음)
│      return;             │   User Space에서만 처리
│                          │
│  syscall(FUTEX_WAIT)     │ ← Slow Path (경합 있음)
└──────────────────────────┘   Kernel 진입
            │
            ↓
Kernel Space:
┌──────────────────────────┐
│  Wait Queue              │
│  [Thread1][Thread2]...   │ ← 대기 중인 스레드들
│                          │
│  FUTEX_WAKE 호출 시      │
│  → 대기 스레드 깨움      │
└──────────────────────────┘
```

### 핵심 시스템 콜

```c
#include <linux/futex.h>
#include <sys/syscall.h>

// Futex Wait: 값이 예상과 같으면 대기
long syscall(SYS_futex,
             int *uaddr,           // futex 변수 주소
             int futex_op,         // FUTEX_WAIT, FUTEX_WAKE 등
             int val,              // 비교할 값
             const struct timespec *timeout,
             int *uaddr2,          // FUTEX_REQUEUE용
             int val3);

// Futex Wake: 대기 중인 스레드 깨우기
long syscall(SYS_futex,
             int *uaddr,
             FUTEX_WAKE,
             int val,              // 깨울 스레드 수
             NULL, NULL, 0);
```

---

## 💻 Futex 기반 Mutex 구현

### 1. 간단한 Futex Mutex

```c
#include <linux/futex.h>
#include <sys/syscall.h>
#include <stdatomic.h>
#include <unistd.h>

typedef struct {
    atomic_int futex_word;  // 0: unlocked, 1: locked (경합 없음), 2: locked (경합 있음)
} futex_mutex_t;

// Futex 시스템 콜 래퍼
static long futex_wait(int *uaddr, int val) {
    return syscall(SYS_futex, uaddr, FUTEX_WAIT_PRIVATE, val, NULL, NULL, 0);
}

static long futex_wake(int *uaddr, int count) {
    return syscall(SYS_futex, uaddr, FUTEX_WAKE_PRIVATE, count, NULL, NULL, 0);
}

// Mutex 초기화
void futex_mutex_init(futex_mutex_t *mutex) {
    atomic_init(&mutex->futex_word, 0);
}

// Mutex Lock
void futex_mutex_lock(futex_mutex_t *mutex) {
    int c;

    // Fast Path: 경합 없이 락 획득 시도
    if ((c = atomic_exchange(&mutex->futex_word, 1)) == 0) {
        return;  // 성공! (User Space만 사용)
    }

    // Slow Path: 경합 있음
    do {
        // 이미 다른 스레드가 대기 중임을 표시
        if (c == 2 || atomic_exchange(&mutex->futex_word, 2) != 0) {
            // Kernel에게 대기 요청
            futex_wait((int*)&mutex->futex_word, 2);
        }
    } while ((c = atomic_exchange(&mutex->futex_word, 2)) != 0);
}

// Mutex Unlock
void futex_mutex_unlock(futex_mutex_t *mutex) {
    // 락 해제
    if (atomic_fetch_sub(&mutex->futex_word, 1) != 1) {
        // 대기 중인 스레드가 있음 (futex_word가 2였음)
        atomic_store(&mutex->futex_word, 0);
        futex_wake((int*)&mutex->futex_word, 1);  // 한 스레드 깨우기
    }
}
```

**동작 과정**:
```
초기 상태: futex_word = 0 (unlocked)

Thread 1: lock()
  1. atomic_exchange(futex_word, 1)
  2. 이전 값이 0 → 즉시 반환 (Fast Path, ~10 cycles)

Thread 2: lock() (Thread 1이 아직 unlock 안함)
  1. atomic_exchange(futex_word, 1)
  2. 이전 값이 1 → 경합 감지
  3. atomic_exchange(futex_word, 2) → "대기자 있음" 표시
  4. futex_wait(futex_word, 2) → Kernel에 대기 (Slow Path)

Thread 1: unlock()
  1. atomic_fetch_sub(futex_word, 1) → 2에서 1로
  2. 이전 값이 2 → 대기자 있음 감지
  3. atomic_store(futex_word, 0)
  4. futex_wake(futex_word, 1) → Thread 2 깨우기
```

### 2. Futex Semaphore

```c
typedef struct {
    atomic_int count;  // 사용 가능한 리소스 수
} futex_semaphore_t;

void futex_sem_init(futex_semaphore_t *sem, int initial_count) {
    atomic_init(&sem->count, initial_count);
}

void futex_sem_wait(futex_semaphore_t *sem) {
    while (1) {
        int c = atomic_load(&sem->count);

        if (c > 0) {
            // Fast Path: count > 0이면 감소 시도
            if (atomic_compare_exchange_weak(&sem->count, &c, c - 1)) {
                return;  // 성공
            }
        } else {
            // Slow Path: count == 0, 대기
            futex_wait((int*)&sem->count, 0);
        }
    }
}

void futex_sem_post(futex_semaphore_t *sem) {
    atomic_fetch_add(&sem->count, 1);
    futex_wake((int*)&sem->count, 1);  // 대기 중인 스레드 하나 깨우기
}
```

### 3. Futex Condition Variable

```c
typedef struct {
    atomic_int futex_word;
    atomic_int waiters;  // 대기 중인 스레드 수
} futex_cond_t;

void futex_cond_init(futex_cond_t *cond) {
    atomic_init(&cond->futex_word, 0);
    atomic_init(&cond->waiters, 0);
}

void futex_cond_wait(futex_cond_t *cond, futex_mutex_t *mutex) {
    atomic_fetch_add(&cond->waiters, 1);

    int seq = atomic_load(&cond->futex_word);

    futex_mutex_unlock(mutex);  // Mutex 해제
    futex_wait((int*)&cond->futex_word, seq);  // 대기
    futex_mutex_lock(mutex);    // 재획득

    atomic_fetch_sub(&cond->waiters, 1);
}

void futex_cond_signal(futex_cond_t *cond) {
    atomic_fetch_add(&cond->futex_word, 1);  // Sequence 번호 증가
    if (atomic_load(&cond->waiters) > 0) {
        futex_wake((int*)&cond->futex_word, 1);
    }
}

void futex_cond_broadcast(futex_cond_t *cond) {
    atomic_fetch_add(&cond->futex_word, 1);
    int waiters = atomic_load(&cond->waiters);
    if (waiters > 0) {
        futex_wake((int*)&cond->futex_word, waiters);  // 모든 스레드 깨우기
    }
}
```

---

## 🔍 Futex 연산 상세

### FUTEX_WAIT

**기능**: futex 값이 예상 값과 같으면 대기

```c
syscall(SYS_futex, &futex_word, FUTEX_WAIT, expected_val, timeout, NULL, 0);
```

**동작**:
1. **Atomic 비교**: `futex_word == expected_val`인지 확인
2. **같으면**: 스레드를 대기 큐에 추가하고 sleep
3. **다르면**: 즉시 반환 (EAGAIN)

**중요**: 비교와 대기가 **원자적(atomic)**으로 수행됨 → Race Condition 방지

### FUTEX_WAKE

**기능**: 대기 중인 스레드 깨우기

```c
syscall(SYS_futex, &futex_word, FUTEX_WAKE, num_to_wake, NULL, NULL, 0);
```

**동작**:
1. 대기 큐에서 최대 `num_to_wake`개 스레드 선택
2. 선택된 스레드를 Runnable 상태로 전환
3. 깨워진 스레드 수 반환

### 기타 Futex 연산

| 연산 | 설명 |
|------|------|
| **FUTEX_REQUEUE** | 대기 스레드를 다른 futex로 이동 (pthread_cond_broadcast 최적화) |
| **FUTEX_CMP_REQUEUE** | 조건부 requeue |
| **FUTEX_WAKE_OP** | Wake + Atomic 연산 (pthread_cond_signal 최적화) |
| **FUTEX_LOCK_PI** | Priority Inheritance 지원 Mutex |
| **FUTEX_TRYLOCK_PI** | Non-blocking PI Lock |

---

## 🔧 커널 내부 구현 상세

### 커널 해시 테이블 구조

```c
// 커널의 futex 대기 큐 관리
struct futex_hash_bucket {
    atomic_t waiters;           // 대기자 수 (최적화용)
    spinlock_t lock;            // 버킷 보호용 락
    struct plist_head chain;    // 우선순위 리스트
};

// 전역 해시 테이블 (크기: 1 << futex_hashshift)
static struct futex_hash_bucket *futex_queues;
```

**해시 함수**:
```c
// futex 주소를 해시 버킷으로 매핑
struct futex_hash_bucket *hash_futex(union futex_key *key) {
    u32 hash = jhash2((u32 *)&key->both.word,
                      sizeof(key->both) / 4,
                      key->both.offset);
    return &futex_queues[hash & (futex_hashsize - 1)];
}
```

**해시 충돌 영향**:
```
문제: 서로 다른 futex 주소가 같은 버킷에 매핑
→ 관련 없는 스레드들이 같은 spinlock 경합
→ False sharing과 유사한 성능 저하

해결: 충분히 큰 해시 테이블 사용 (보통 256~1024 버킷)
```

### Futex Key 구조

```c
// 커널에서 futex를 식별하는 키
union futex_key {
    struct {
        u64 i_seq;              // inode sequence (파일 기반)
        unsigned long pgoff;    // 페이지 오프셋
        unsigned int offset;    // 페이지 내 오프셋
    } shared;                   // 프로세스 간 공유 futex

    struct {
        union {
            struct mm_struct *mm;
            u64 __tmp;
        };
        unsigned long address;  // 가상 주소
        unsigned int offset;    // 페이지 내 오프셋
    } private;                  // 프로세스 내 futex

    struct {
        u64 ptr;
        unsigned long word;
        unsigned int offset;
    } both;
};
```

### FUTEX_WAIT 커널 구현

```c
// kernel/futex.c (단순화)
static int futex_wait(u32 __user *uaddr, unsigned int flags,
                      u32 val, ktime_t *abs_time) {
    struct futex_hash_bucket *hb;
    struct futex_q q = FUTEX_Q_INIT;
    int ret;

    // 1. Futex 키 생성
    ret = get_futex_key(uaddr, flags, &q.key);
    if (ret)
        return ret;

    // 2. 해시 버킷 찾기 및 락
    hb = queue_lock(&q);

    // 3. 사용자 공간 값 확인 (원자적으로)
    ret = get_futex_value_locked(&uval, uaddr);
    if (ret)
        goto out_unlock;

    // 4. 값이 다르면 즉시 반환 (EAGAIN)
    if (uval != val) {
        ret = -EAGAIN;
        goto out_unlock;
    }

    // 5. 대기 큐에 추가
    futex_wait_queue(&q, hb);

    // 6. 스케줄 아웃 (sleep)
    if (!signal_pending(current))
        schedule();

    // 7. 깨어남
    return ret;

out_unlock:
    queue_unlock(&q, hb);
    return ret;
}
```

### FUTEX_WAKE 커널 구현

```c
static int futex_wake(u32 __user *uaddr, unsigned int flags,
                      int nr_wake) {
    struct futex_hash_bucket *hb;
    struct futex_q *q, *tmp;
    union futex_key key;
    int ret = 0;

    // 1. Futex 키 생성
    get_futex_key(uaddr, flags, &key);

    // 2. 해시 버킷 락
    hb = hash_futex(&key);
    spin_lock(&hb->lock);

    // 3. 대기 큐에서 매칭되는 스레드 찾기
    plist_for_each_entry_safe(q, tmp, &hb->chain, list) {
        if (match_futex(&q->key, &key)) {
            // 4. 깨우기
            wake_futex(q);
            if (++ret >= nr_wake)
                break;
        }
    }

    spin_unlock(&hb->lock);
    return ret;
}
```

### FUTEX_PRIVATE_FLAG 최적화

```c
// Private futex (같은 프로세스 내):
// - 가상 주소만으로 식별 가능
// - 페이지 테이블 조회 불필요
// - ~30% 더 빠름

futex(addr, FUTEX_WAIT_PRIVATE, val);  // 빠름
futex(addr, FUTEX_WAIT, val);          // 느림 (공유 가능)
```

### Futex vs 다른 동기화 비용

```
동기화 프리미티브 비용 비교 (uncontended):

┌────────────────────────────────────────────┐
│ Atomic CAS:             ~10-20 cycles     │
│ Futex (fast path):      ~15-25 cycles     │
│ pthread_mutex:          ~20-30 cycles     │
│ Futex (slow path):      ~1000+ cycles     │
│ System V Semaphore:     ~500+ cycles      │
└────────────────────────────────────────────┘

→ Fast path가 중요한 이유:
   90%+ 경우가 uncontended
   → 대부분 user-space에서만 처리
```

---

## 📊 성능 분석

### 벤치마크: Futex Mutex vs pthread_mutex

```c
// 벤치마크 코드
#include <pthread.h>
#include <time.h>
#include <stdio.h>

#define ITERATIONS 1000000

void benchmark_futex_mutex() {
    futex_mutex_t mutex;
    futex_mutex_init(&mutex);

    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < ITERATIONS; i++) {
        futex_mutex_lock(&mutex);
        // Critical section (empty)
        futex_mutex_unlock(&mutex);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("Futex Mutex: %.3f seconds (%.0f ns/op)\n",
           elapsed, elapsed * 1e9 / ITERATIONS);
}

void benchmark_pthread_mutex() {
    pthread_mutex_t mutex;
    pthread_mutex_init(&mutex, NULL);

    struct timespec start, end;
    clock_gettime(CLOCK_MONOTONIC, &start);

    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&mutex);
        // Critical section (empty)
        pthread_mutex_unlock(&mutex);
    }

    clock_gettime(CLOCK_MONOTONIC, &end);
    double elapsed = (end.tv_sec - start.tv_sec) +
                     (end.tv_nsec - start.tv_nsec) / 1e9;

    printf("pthread_mutex: %.3f seconds (%.0f ns/op)\n",
           elapsed, elapsed * 1e9 / ITERATIONS);

    pthread_mutex_destroy(&mutex);
}
```

**예상 결과** (경합 없음):
```
Futex Mutex:   0.010 seconds (10 ns/op)   ← User space atomic만
pthread_mutex: 0.012 seconds (12 ns/op)   ← pthread도 내부적으로 futex 사용
```

**관찰**: 현대 pthread 구현은 내부적으로 futex 사용 → 성능 유사

---

## 🎯 실제 사용 사례

### glibc pthread 구현

glibc의 pthread_mutex는 내부적으로 futex 사용:

```c
// glibc/nptl/pthread_mutex_lock.c (단순화)
int pthread_mutex_lock(pthread_mutex_t *mutex) {
    // Fast path: 경합 없이 락 획득 시도
    if (__glibc_likely(atomic_compare_exchange_weak_acquire(&mutex->__data.__lock,
                                                             &LLL_LOCK_INITIALIZER,
                                                             LLL_LOCK_INITIALIZER))) {
        return 0;
    }

    // Slow path: futex를 이용한 대기
    return __pthread_mutex_lock_full(mutex);
}
```

### Go 언어 runtime

Go의 sync.Mutex도 Linux에서 futex 사용:

```go
// runtime/lock_futex.go
func lock(l *mutex) {
    // Fast path
    if atomic.Casuintptr(&l.key, 0, locked) {
        return
    }

    // Slow path: futex wait
    futexsleep(&l.key, locked, -1)
}
```

### Rust std::sync::Mutex

```rust
// std/src/sys/unix/locks/futex_mutex.rs
impl Mutex {
    pub fn lock(&self) {
        if self.futex.compare_exchange(0, 1, Acquire, Relaxed).is_ok() {
            return;  // Fast path
        }
        self.lock_contended();  // Slow path: futex
    }
}
```

---

## ⚠️ 주의사항

### 1. ABA 문제

```c
// ❌ 위험: ABA 문제
int futex_word = 0;

// Thread 1
if (futex_word == 0) {
    futex_wait(&futex_word, 0);  // 대기
}

// Thread 2
futex_word = 1;  // 깨움
futex_word = 0;  // 다시 0으로

// Thread 1은 여전히 대기 중!
```

**해결**: Sequence number 사용

```c
atomic_int seq = 0;

// Wait
int current_seq = atomic_load(&seq);
futex_wait(&seq, current_seq);

// Signal
atomic_fetch_add(&seq, 1);  // Sequence 증가
futex_wake(&seq, 1);
```

### 2. Spurious Wakeup

Futex도 spurious wakeup 발생 가능 → **항상 루프에서 조건 재확인**

```c
// ✅ 올바른 패턴
while (!condition) {
    futex_wait(&futex_word, expected_val);
}
```

### 3. 플랫폼 의존성

- **Linux 전용**: futex는 Linux 커널 2.6+에서만 사용 가능
- **크로스 플랫폼**: Windows (WaitOnAddress), macOS (ulock) 등 다른 API 사용

---

## 🔄 플랫폼별 유사 메커니즘

| 플랫폼 | 메커니즘 | API |
|--------|----------|-----|
| **Linux** | Futex | `syscall(SYS_futex, ...)` |
| **Windows** | WaitOnAddress | `WaitOnAddress()`, `WakeByAddressSingle()` |
| **macOS/iOS** | ulock | `__ulock_wait()`, `__ulock_wake()` |
| **FreeBSD** | umtx | `_umtx_op()` |

**크로스 플랫폼 대안**: C11 `atomic_wait` / `atomic_notify` (C++20 `std::atomic::wait`)

```cpp
#include <atomic>

std::atomic<int> futex_word{0};

// Wait
futex_word.wait(0);  // 0이 아닐 때까지 대기

// Notify
futex_word.store(1);
futex_word.notify_one();  // 또는 notify_all()
```

---

## 📚 고급 주제

### Priority Inheritance (FUTEX_LOCK_PI)

Priority Inversion 문제 해결:

```c
// Priority Inheritance Mutex
syscall(SYS_futex, &futex_word, FUTEX_LOCK_PI, 0, NULL, NULL, 0);
```

**동작**:
- 낮은 우선순위 스레드가 락을 보유 중
- 높은 우선순위 스레드가 대기
- **커널이 자동으로 낮은 우선순위 스레드의 우선순위를 높임**

### FUTEX_REQUEUE (Condition Variable 최적화)

pthread_cond_broadcast의 Thundering Herd 문제 해결:

```c
// 기존 방식: 모든 스레드 깨우기 → 하나만 mutex 획득, 나머지는 다시 sleep
futex_wake(&cond_futex, INT_MAX);

// 최적화: 하나만 깨우고 나머지는 mutex futex로 이동
syscall(SYS_futex, &cond_futex, FUTEX_REQUEUE,
        1,              // 깨울 스레드 수
        INT_MAX,        // requeue할 스레드 수
        &mutex_futex,   // 이동할 대상
        0);
```

---

## 🔗 관련 문서

- [Mutex / Lock](./01-mutex-lock.md) - 고수준 Mutex
- [Condition Variable](./03-condition-variable.md) - Futex 기반 CV
- [Atomic Operations](./04-atomic-operations.md) - Futex의 Fast Path
- [플랫폼 차이](../10-platform-differences/README.md) - Windows vs Linux

---

## 📖 참고 자료

### 공식 문서
- [futex(2) man page](https://man7.org/linux/man-pages/man2/futex.2.html)
- [futex(7) man page](https://man7.org/linux/man-pages/man7/futex.7.html)
- [Futex Requeue PI](https://lwn.net/Articles/685769/)

### 논문 및 기술 자료
- Franke, Russell, Kirkwood (2002). "Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux"
- Drepper, Ulrich (2011). "Futexes Are Tricky" - Red Hat

### 소스 코드
- [Linux Kernel futex.c](https://github.com/torvalds/linux/blob/master/kernel/futex.c)
- [glibc NPTL implementation](https://sourceware.org/git/?p=glibc.git;a=tree;f=nptl)

---

## 💡 요약

### Futex의 핵심 장점
1. ✅ **하이브리드 접근**: User space (빠름) + Kernel space (필요시만)
2. ✅ **뛰어난 성능**: 경합 없을 때 ~10배 빠름
3. ✅ **유연성**: Mutex, Semaphore, CV 등 모든 동기화 도구의 기반

### 사용 권장
- **직접 사용**: 고급 사용자, 커스텀 동기화 프리미티브 구현
- **간접 사용**: pthread, C++11 std::mutex 등 표준 라이브러리 (내부적으로 futex 사용)

**결론**: 대부분의 경우 pthread나 std::mutex 사용 권장. Futex는 저수준 이해와 극한 최적화가 필요한 경우에만 직접 사용.

---

*Futex는 현대 Linux 동기화의 핵심입니다. pthread mutex도 내부적으로 futex를 사용합니다!*
