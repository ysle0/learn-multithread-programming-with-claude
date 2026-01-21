# Windows 동기화 프리미티브 (WaitOnAddress & CRITICAL_SECTION)

## 📌 개요

Windows는 **하이브리드 동기화 메커니즘**을 통해 유저 스페이스와 커널 스페이스를 효율적으로 결합합니다. 이 문서에서는 Windows의 핵심 저수준 동기화 프리미티브를 다룹니다:

1. **WaitOnAddress** (Windows 8+): Linux futex와 동등한 기능
2. **CRITICAL_SECTION**: Windows의 대표적인 경량 Mutex
3. **SRW Lock**: Slim Reader/Writer Lock (읽기/쓰기 분리)

**핵심 아이디어**: 경합이 없을 때는 유저 스페이스에서만 동작(빠름), 경합이 있을 때만 커널 호출(느림)

---

## 🎯 등장 배경

### 전통적인 동기화 메커니즘의 문제

**Mutex, Semaphore (CreateMutex, CreateSemaphore)**:
```cpp
// 매번 커널 호출 필요
WaitForSingleObject(hMutex, INFINITE);  // 시스템 콜 (항상 느림)
// Critical Section
ReleaseMutex(hMutex);                   // 시스템 콜 (항상 느림)
```

**문제점**:
- ⚠️ 경합이 없어도 항상 커널 모드 진입
- ⚠️ Context Switching 오버헤드
- ⚠️ 시스템 콜 비용: 수백 cycles
- ⚠️ 커널 오브젝트 핸들 관리 오버헤드

### Windows 하이브리드 동기화의 해결책

```
경합 없음 (Fast Path):
    User Space에서만 Interlocked 연산 → 매우 빠름 (10-20 cycles)

경합 있음 (Slow Path):
    Kernel Space 진입 → 스레드 대기/깨우기
```

**성능 비교**:
| 상황 | Mutex (커널 오브젝트) | CRITICAL_SECTION |
|------|---------------------|-----------------|
| **경합 없음** | ~200 cycles (시스템 콜) | ~10 cycles (Interlocked) |
| **경합 있음** | ~200 cycles | ~200 cycles |

→ **경합이 드문 경우 20배 빠름!**

---

## 🏗️ Windows 동기화 아키텍처

### 1. CRITICAL_SECTION 구조

```
User Space:
┌──────────────────────────────┐
│  CRITICAL_SECTION cs;        │
│  cs.LockCount = -1;          │ ← Interlocked 변수
│                              │
│  if (InterlockedIncrement()) │ ← Fast Path (경합 없음)
│      return;                 │   User Space에서만 처리
│                              │
│  WaitForSingleObject(Event)  │ ← Slow Path (경합 있음)
└──────────────────────────────┘   Kernel 진입
            │
            ↓
Kernel Space:
┌──────────────────────────────┐
│  Event Object                │
│  Wait Queue                  │ ← 대기 중인 스레드들
│  [Thread1][Thread2]...       │
│                              │
│  SetEvent 호출 시            │
│  → 대기 스레드 깨움          │
└──────────────────────────────┘
```

### 2. WaitOnAddress API (Windows 8+)

**Linux futex와 동등한 Windows API**:

```cpp
#include <synchapi.h>

// 값이 예상과 같으면 대기
BOOL WaitOnAddress(
    volatile VOID *Address,       // 대기할 주소
    PVOID CompareAddress,         // 비교할 값의 주소
    SIZE_T AddressSize,           // 1, 2, 4, 8 bytes
    DWORD dwMilliseconds          // Timeout (INFINITE 가능)
);

// 대기 중인 스레드 하나 깨우기
VOID WakeByAddressSingle(PVOID Address);

// 대기 중인 모든 스레드 깨우기
VOID WakeByAddressAll(PVOID Address);
```

---

## 💻 CRITICAL_SECTION 사용법

### 1. 기본 사용

```cpp
#include <windows.h>
#include <stdio.h>

CRITICAL_SECTION cs;
int shared_counter = 0;

DWORD WINAPI ThreadFunc(LPVOID lpParam) {
    for (int i = 0; i < 100000; i++) {
        EnterCriticalSection(&cs);  // Lock
        shared_counter++;
        LeaveCriticalSection(&cs);  // Unlock
    }
    return 0;
}

int main() {
    // 초기화
    InitializeCriticalSection(&cs);

    // 스레드 생성
    HANDLE threads[4];
    for (int i = 0; i < 4; i++) {
        threads[i] = CreateThread(NULL, 0, ThreadFunc, NULL, 0, NULL);
    }

    // 대기
    WaitForMultipleObjects(4, threads, TRUE, INFINITE);

    printf("Counter: %d\n", shared_counter);  // 400000

    // 정리
    DeleteCriticalSection(&cs);

    for (int i = 0; i < 4; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

### 2. CRITICAL_SECTION 최적화 옵션

```cpp
// Spin Count 설정 (멀티코어 시스템에서 유용)
InitializeCriticalSectionAndSpinCount(&cs, 4000);

// 또는 나중에 설정
SetCriticalSectionSpinCount(&cs, 4000);
```

**Spin Count 동작**:
- 락 획득 실패 시 즉시 커널 모드로 가지 않고 먼저 **spin**
- Spin Count만큼 busy-waiting 후에도 실패하면 커널 대기
- **멀티코어에서만 유효** (단일 코어에서는 0 권장)

### 3. Try-Enter (Non-blocking)

```cpp
BOOL TryEnterCriticalSection(LPCRITICAL_SECTION lpCriticalSection);

// 사용 예시
if (TryEnterCriticalSection(&cs)) {
    // 락 획득 성공
    shared_data++;
    LeaveCriticalSection(&cs);
} else {
    // 락 획득 실패, 다른 작업 수행
    printf("Lock busy, doing other work...\n");
}
```

---

## 💻 WaitOnAddress 기반 커스텀 동기화 구현

### 1. WaitOnAddress 기반 Mutex

```cpp
#include <windows.h>
#include <stdio.h>

typedef struct {
    volatile LONG locked;  // 0: unlocked, 1: locked
} WOAMutex;

void WOAMutex_Init(WOAMutex *mutex) {
    mutex->locked = 0;
}

void WOAMutex_Lock(WOAMutex *mutex) {
    // Fast Path: 경합 없이 락 획득 시도
    while (InterlockedCompareExchange(&mutex->locked, 1, 0) != 0) {
        // Slow Path: 경합 있음, 대기
        LONG expected = 1;
        WaitOnAddress(&mutex->locked, &expected, sizeof(LONG), INFINITE);
    }
}

void WOAMutex_Unlock(WOAMutex *mutex) {
    InterlockedExchange(&mutex->locked, 0);
    WakeByAddressSingle((PVOID)&mutex->locked);
}

// 사용 예시
WOAMutex my_mutex;

DWORD WINAPI Worker(LPVOID lpParam) {
    for (int i = 0; i < 1000; i++) {
        WOAMutex_Lock(&my_mutex);
        // Critical section
        WOAMutex_Unlock(&my_mutex);
    }
    return 0;
}
```

**동작 과정**:
```
초기 상태: locked = 0 (unlocked)

Thread 1: Lock()
  1. InterlockedCompareExchange(&locked, 1, 0)
  2. 이전 값이 0 → 즉시 반환 (Fast Path, ~10 cycles)

Thread 2: Lock() (Thread 1이 아직 unlock 안함)
  1. InterlockedCompareExchange(&locked, 1, 0)
  2. 이전 값이 1 → 경합 감지
  3. WaitOnAddress(&locked, &expected=1, ...) → Kernel에 대기

Thread 1: Unlock()
  1. InterlockedExchange(&locked, 0)
  2. WakeByAddressSingle() → Thread 2 깨우기
```

### 2. WaitOnAddress 기반 Semaphore

```cpp
typedef struct {
    volatile LONG count;
} WOASemaphore;

void WOASem_Init(WOASemaphore *sem, LONG initial_count) {
    sem->count = initial_count;
}

void WOASem_Wait(WOASemaphore *sem) {
    while (1) {
        LONG current = sem->count;

        if (current > 0) {
            // Fast Path: count > 0이면 감소 시도
            if (InterlockedCompareExchange(&sem->count, current - 1, current) == current) {
                return;  // 성공
            }
        } else {
            // Slow Path: count == 0, 대기
            LONG expected = 0;
            WaitOnAddress(&sem->count, &expected, sizeof(LONG), INFINITE);
        }
    }
}

void WOASem_Post(WOASemaphore *sem) {
    InterlockedIncrement(&sem->count);
    WakeByAddressSingle((PVOID)&sem->count);
}
```

### 3. WaitOnAddress 기반 Condition Variable

```cpp
typedef struct {
    volatile LONG seq;      // Sequence number
    volatile LONG waiters;  // 대기 중인 스레드 수
} WOACondVar;

void WOACond_Init(WOACondVar *cv) {
    cv->seq = 0;
    cv->waiters = 0;
}

void WOACond_Wait(WOACondVar *cv, CRITICAL_SECTION *cs) {
    InterlockedIncrement(&cv->waiters);

    LONG current_seq = cv->seq;

    LeaveCriticalSection(cs);  // Mutex 해제
    WaitOnAddress(&cv->seq, &current_seq, sizeof(LONG), INFINITE);
    EnterCriticalSection(cs);   // 재획득

    InterlockedDecrement(&cv->waiters);
}

void WOACond_Signal(WOACondVar *cv) {
    InterlockedIncrement(&cv->seq);
    if (cv->waiters > 0) {
        WakeByAddressSingle((PVOID)&cv->seq);
    }
}

void WOACond_Broadcast(WOACondVar *cv) {
    InterlockedIncrement(&cv->seq);
    if (cv->waiters > 0) {
        WakeByAddressAll((PVOID)&cv->seq);
    }
}
```

---

## 🔍 SRW Lock (Slim Reader/Writer Lock)

**Windows Vista+의 경량 RWLock**:

```cpp
#include <windows.h>

SRWLOCK srw = SRWLOCK_INIT;
int shared_data = 0;

// Reader (공유 락)
void Reader() {
    AcquireSRWLockShared(&srw);
    int value = shared_data;  // 여러 Reader 동시 가능
    ReleaseSRWLockShared(&srw);
}

// Writer (독점 락)
void Writer() {
    AcquireSRWLockExclusive(&srw);
    shared_data++;  // 독점 접근
    ReleaseSRWLockExclusive(&srw);
}

// 초기화는 정적 또는 동적
SRWLOCK srw1 = SRWLOCK_INIT;  // 정적 초기화
// 또는
SRWLOCK srw2;
InitializeSRWLock(&srw2);  // 동적 초기화
```

**장점**:
- ✅ **sizeof(void\*)** 크기 (32bit: 4바이트, 64bit: 8바이트)
- ✅ **초기화 불필요** (SRWLOCK_INIT 사용 시)
- ✅ **삭제 불필요** (커널 오브젝트 없음)
- ✅ **읽기 병렬화** (여러 Reader 동시 진입)

---

## 📊 성능 분석

### 벤치마크: CRITICAL_SECTION vs Mutex vs WaitOnAddress

```cpp
#include <windows.h>
#include <stdio.h>

#define ITERATIONS 1000000

void benchmark_critical_section() {
    CRITICAL_SECTION cs;
    InitializeCriticalSection(&cs);

    LARGE_INTEGER start, end, freq;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&start);

    for (int i = 0; i < ITERATIONS; i++) {
        EnterCriticalSection(&cs);
        // Critical section (empty)
        LeaveCriticalSection(&cs);
    }

    QueryPerformanceCounter(&end);
    double elapsed_ms = (double)(end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
    double ns_per_op = elapsed_ms * 1e6 / ITERATIONS;

    printf("CRITICAL_SECTION: %.3f ms (%.0f ns/op)\n", elapsed_ms, ns_per_op);
    DeleteCriticalSection(&cs);
}

void benchmark_mutex() {
    HANDLE hMutex = CreateMutex(NULL, FALSE, NULL);

    LARGE_INTEGER start, end, freq;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&start);

    for (int i = 0; i < ITERATIONS; i++) {
        WaitForSingleObject(hMutex, INFINITE);
        // Critical section (empty)
        ReleaseMutex(hMutex);
    }

    QueryPerformanceCounter(&end);
    double elapsed_ms = (double)(end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
    double ns_per_op = elapsed_ms * 1e6 / ITERATIONS;

    printf("Mutex (Kernel):   %.3f ms (%.0f ns/op)\n", elapsed_ms, ns_per_op);
    CloseHandle(hMutex);
}

void benchmark_waitonaddress() {
    WOAMutex mutex;
    WOAMutex_Init(&mutex);

    LARGE_INTEGER start, end, freq;
    QueryPerformanceFrequency(&freq);
    QueryPerformanceCounter(&start);

    for (int i = 0; i < ITERATIONS; i++) {
        WOAMutex_Lock(&mutex);
        // Critical section (empty)
        WOAMutex_Unlock(&mutex);
    }

    QueryPerformanceCounter(&end);
    double elapsed_ms = (double)(end.QuadPart - start.QuadPart) * 1000.0 / freq.QuadPart;
    double ns_per_op = elapsed_ms * 1e6 / ITERATIONS;

    printf("WaitOnAddress:    %.3f ms (%.0f ns/op)\n", elapsed_ms, ns_per_op);
}

int main() {
    benchmark_critical_section();
    benchmark_mutex();
    benchmark_waitonaddress();
    return 0;
}
```

**예상 결과** (경합 없음):
```
CRITICAL_SECTION: 10.5 ms (10 ns/op)   ← 가장 빠름 (Interlocked)
WaitOnAddress:    11.2 ms (11 ns/op)   ← CRITICAL_SECTION과 유사
Mutex (Kernel):   180.0 ms (180 ns/op) ← 항상 커널 호출
```

**관찰**:
- CRITICAL_SECTION이 가장 빠름 (최적화된 구현)
- WaitOnAddress는 유연하지만 약간 느림 (수동 구현)
- Mutex는 항상 커널 호출 → 매우 느림

---

## 🎯 실제 사용 사례

### Windows CRT (C Runtime) 구현

Windows CRT의 _lock/_unlock은 내부적으로 CRITICAL_SECTION 사용:

```cpp
// CRT 내부 구현 (단순화)
static CRITICAL_SECTION _lock_table[_MAX_LOCK];

void __cdecl _lock(int locknum) {
    EnterCriticalSection(&_lock_table[locknum]);
}

void __cdecl _unlock(int locknum) {
    LeaveCriticalSection(&_lock_table[locknum]);
}
```

### .NET Framework

.NET의 `lock` 키워드는 Monitor.Enter/Exit 사용 → 내부적으로 CRITICAL_SECTION:

```csharp
// C# 코드
lock (myObject) {
    // Critical section
}

// 컴파일 후 (IL)
Monitor.Enter(myObject);
try {
    // Critical section
} finally {
    Monitor.Exit(myObject);
}
```

### Win32 애플리케이션

```cpp
// 전역 변수 보호
CRITICAL_SECTION g_cs;
std::vector<int> g_shared_data;

void Initialize() {
    InitializeCriticalSectionAndSpinCount(&g_cs, 4000);
}

void AddData(int value) {
    EnterCriticalSection(&g_cs);
    g_shared_data.push_back(value);
    LeaveCriticalSection(&g_cs);
}
```

---

## ⚠️ 주의사항

### 1. CRITICAL_SECTION 재진입 (Recursive)

**CRITICAL_SECTION은 재귀 락 지원**:

```cpp
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);

void Function() {
    EnterCriticalSection(&cs);
    // ...
    EnterCriticalSection(&cs);  // ✅ OK (같은 스레드)
    // ...
    LeaveCriticalSection(&cs);
    LeaveCriticalSection(&cs);
}
```

**주의**: Enter/Leave 횟수가 일치해야 함

### 2. 삭제 시점

```cpp
// ❌ 잘못된 예: 다른 스레드가 사용 중
DeleteCriticalSection(&cs);

// ✅ 올바른 예: 모든 스레드 종료 후
WaitForMultipleObjects(num_threads, threads, TRUE, INFINITE);
DeleteCriticalSection(&cs);
```

### 3. Deadlock 방지

```cpp
// ❌ Deadlock 위험
void Thread1() {
    EnterCriticalSection(&cs1);
    EnterCriticalSection(&cs2);
    // ...
    LeaveCriticalSection(&cs2);
    LeaveCriticalSection(&cs1);
}

void Thread2() {
    EnterCriticalSection(&cs2);  // 반대 순서!
    EnterCriticalSection(&cs1);
    // ...
    LeaveCriticalSection(&cs1);
    LeaveCriticalSection(&cs2);
}

// ✅ 해결: 항상 같은 순서
void Thread1() {
    EnterCriticalSection(&cs1);
    EnterCriticalSection(&cs2);
    // ...
}

void Thread2() {
    EnterCriticalSection(&cs1);  // 같은 순서
    EnterCriticalSection(&cs2);
    // ...
}
```

### 4. WaitOnAddress 제약사항

**Windows 8 이상 필요**:
```cpp
// Windows 7 이하에서는 사용 불가
#if WINVER >= 0x0602  // Windows 8+
    WaitOnAddress(&var, &expected, sizeof(var), INFINITE);
#else
    // Fallback: Event 또는 CRITICAL_SECTION 사용
#endif
```

### 5. Spurious Wakeup

WaitOnAddress도 spurious wakeup 발생 가능 → **항상 루프에서 조건 재확인**:

```cpp
// ✅ 올바른 패턴
while (condition != expected_value) {
    WaitOnAddress(&condition, &expected_value, sizeof(condition), INFINITE);
}
```

---

## 🔄 플랫폼별 유사 메커니즘

| 플랫폼 | 메커니즘 | API |
|--------|----------|-----|
| **Windows** | WaitOnAddress | `WaitOnAddress()`, `WakeByAddressSingle()` |
| **Windows** | CRITICAL_SECTION | `EnterCriticalSection()`, `LeaveCriticalSection()` |
| **Linux** | Futex | `syscall(SYS_futex, ...)` |
| **macOS/iOS** | ulock | `__ulock_wait()`, `__ulock_wake()` |
| **FreeBSD** | umtx | `_umtx_op()` |

**크로스 플랫폼 대안**: C++20 `std::atomic::wait` / `std::atomic::notify_one`

```cpp
#include <atomic>

std::atomic<int> flag{0};

// Wait
flag.wait(0);  // 0이 아닐 때까지 대기

// Notify
flag.store(1);
flag.notify_one();  // 또는 notify_all()
```

---

## 📊 동기화 메커니즘 선택 가이드

### Windows 동기화 프리미티브 비교

| 메커니즘 | Fast Path | Slow Path | 재진입 | 크기 | 삭제 필요 | 용도 |
|---------|-----------|-----------|--------|------|----------|------|
| **CRITICAL_SECTION** | ~10ns | ~200ns | ✅ | ~24바이트 | ✅ | 일반적 동기화 (권장) |
| **SRW Lock** | ~10ns | ~200ns | ❌ | 4-8바이트 | ❌ | 읽기 위주 (90%+) |
| **Mutex (커널)** | ~180ns | ~200ns | ❌ | 핸들 | ✅ | 프로세스 간 동기화 |
| **Semaphore (커널)** | ~180ns | ~200ns | - | 핸들 | ✅ | 리소스 카운팅 |
| **Event (커널)** | ~180ns | ~200ns | - | 핸들 | ✅ | 이벤트 알림 |
| **WaitOnAddress** | ~11ns | ~200ns | ❌ | 사용자 정의 | ❌ | 커스텀 동기화 (Win8+) |

### 선택 기준

```
일반적인 상호 배제?
    └─→ CRITICAL_SECTION (가장 권장)

읽기가 대부분 (>90%)?
    └─→ SRW Lock (Shared/Exclusive)

프로세스 간 동기화?
    └─→ Named Mutex / Named Semaphore

이벤트 알림?
    └─→ Event Object (Manual/Auto Reset)

커스텀 동기화 구현?
    └─→ WaitOnAddress (Windows 8+)

최소 메모리?
    └─→ SRW Lock (4-8바이트)
```

---

## 📚 고급 주제

### 1. Condition Variable (Windows Vista+)

Windows 기본 제공 Condition Variable:

```cpp
#include <windows.h>

CRITICAL_SECTION cs;
CONDITION_VARIABLE cv;

void Producer() {
    EnterCriticalSection(&cs);
    // 데이터 생성
    WakeConditionVariable(&cv);  // 또는 WakeAllConditionVariable
    LeaveCriticalSection(&cs);
}

void Consumer() {
    EnterCriticalSection(&cs);
    while (!data_ready) {
        SleepConditionVariableCS(&cv, &cs, INFINITE);
    }
    // 데이터 소비
    LeaveCriticalSection(&cs);
}

// 초기화
InitializeCriticalSection(&cs);
InitializeConditionVariable(&cv);  // 삭제 불필요
```

### 2. Interlocked API (Atomic Operations)

```cpp
#include <windows.h>

volatile LONG counter = 0;

// Atomic Increment
InterlockedIncrement(&counter);

// Atomic Decrement
InterlockedDecrement(&counter);

// Atomic Add
InterlockedAdd(&counter, 5);

// Atomic Exchange
LONG old_value = InterlockedExchange(&counter, 100);

// Atomic Compare-And-Swap
LONG expected = 10;
LONG new_value = 20;
InterlockedCompareExchange(&counter, new_value, expected);

// 64-bit 버전
volatile LONG64 counter64 = 0;
InterlockedIncrement64(&counter64);
```

### 3. Memory Barrier

```cpp
// Compiler Barrier (컴파일러 재배치 방지)
_ReadWriteBarrier();

// Full Memory Barrier (CPU + 컴파일러)
MemoryBarrier();

// Acquire/Release 시맨틱 (Interlocked에 내장)
InterlockedExchangeAcquire(&var, value);
InterlockedExchangeRelease(&var, value);
```

---

## 🔗 관련 문서

- [Mutex / Lock](./01-mutex-lock.md) - 고수준 Mutex
- [Condition Variable](./03-condition-variable.md) - CV 사용법
- [Atomic Operations](./04-atomic-operations.md) - Interlocked API
- [플랫폼 차이](../10-platform-differences/README.md) - Windows vs Linux

---

## 📖 참고 자료

### 공식 문서
- [CRITICAL_SECTION](https://docs.microsoft.com/en-us/windows/win32/sync/critical-section-objects)
- [WaitOnAddress](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitonaddress)
- [SRW Locks](https://docs.microsoft.com/en-us/windows/win32/sync/slim-reader-writer--srw--locks)
- [Interlocked Functions](https://docs.microsoft.com/en-us/windows/win32/sync/interlocked-variable-access)

### 기술 자료
- Richter, Jeffrey (2012). "Windows via C/C++" - Microsoft Press
- Duffy, Joe (2008). "Concurrent Programming on Windows" - Addison-Wesley

### 블로그 및 아티클
- [The Old New Thing - Raymond Chen](https://devblogs.microsoft.com/oldnewthing/)
- [Preshing on Programming - Synchronization](https://preshing.com/archives/)

---

## 💡 요약

### Windows 동기화의 핵심 장점

1. ✅ **하이브리드 접근**: User space (빠름) + Kernel space (필요시만)
2. ✅ **뛰어난 성능**: 경합 없을 때 ~20배 빠름
3. ✅ **다양한 선택지**: CRITICAL_SECTION, SRW, WaitOnAddress 등
4. ✅ **재진입 지원**: CRITICAL_SECTION은 recursive lock

### 사용 권장

| 시나리오 | 권장 메커니즘 |
|---------|-------------|
| **일반적 동기화** | CRITICAL_SECTION |
| **읽기 위주** | SRW Lock |
| **프로세스 간** | Named Mutex/Semaphore |
| **커스텀 구현** | WaitOnAddress (Win8+) |
| **이벤트** | CONDITION_VARIABLE 또는 Event |

**결론**: 대부분의 경우 **CRITICAL_SECTION** 사용 권장. WaitOnAddress는 특수한 커스텀 동기화가 필요한 경우에만 사용.

---

*Windows는 강력한 하이브리드 동기화 메커니즘을 제공합니다. 상황에 맞는 도구를 선택하세요!*
