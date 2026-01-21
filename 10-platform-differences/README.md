# Windows vs POSIX 플랫폼 차이

## 📌 개요

멀티스레드 프로그래밍은 플랫폼마다 서로 다른 API와 동작 방식을 가집니다. 이 섹션에서는 **Windows API**와 **POSIX (Portable Operating System Interface)**의 차이점을 상세히 비교하여, 크로스 플랫폼 개발 시 주의사항과 최선의 방법을 제시합니다.

### 🎯 왜 중요한가?

- ✅ **크로스 플랫폼 개발**: Linux/macOS와 Windows에서 동일하게 동작하는 코드 작성
- ✅ **성능 최적화**: 플랫폼별 최적의 API 선택
- ✅ **이식성**: 기존 POSIX 코드를 Windows로 포팅 (또는 그 반대)
- ✅ **디버깅**: 플랫폼별 동작 차이로 인한 버그 이해

---

## 🔍 핵심 차이점 요약

| 특성 | Windows | POSIX (Linux/macOS) |
|------|---------|---------------------|
| **스레드 API** | Win32 API (CreateThread) | pthreads (pthread_create) |
| **동기화 객체** | Critical Section, Mutex, Event, Semaphore | Mutex, Condition Variable, Semaphore |
| **철학** | 리소스 핸들 기반 | 파일 디스크립터 유사 개념 |
| **스레드 종료** | ExitThread, TerminateThread | pthread_exit, pthread_cancel |
| **TLS** | TlsAlloc/TlsGetValue | pthread_key_create |
| **네이밍** | Win32 스타일 (CamelCase) | POSIX 스타일 (snake_case) |
| **에러 처리** | GetLastError() | errno |
| **이식성** | Windows 전용 | Linux, macOS, BSD, Unix 등 |

---

## 📚 문서 목록

### [01. 스레드 생성 및 관리](./01-thread-creation.md)
**핵심 비교**: CreateThread vs pthread_create

**다루는 내용**:
- 스레드 생성 API 비교
- 스레드 ID와 핸들
- 스레드 종료 및 정리
- 스레드 속성 (우선순위, 스택 크기)
- Detached vs Joinable 스레드

**코드 예시**:
```cpp
// Windows
HANDLE hThread = CreateThread(NULL, 0, ThreadFunc, pData, 0, &dwThreadId);
WaitForSingleObject(hThread, INFINITE);
CloseHandle(hThread);

// POSIX
pthread_t thread;
pthread_create(&thread, NULL, ThreadFunc, pData);
pthread_join(thread, NULL);
```

---

### [02. 동기화 프리미티브](./02-synchronization-primitives.md)
**핵심 비교**: Critical Section, Mutex, Event vs pthread_mutex, pthread_cond

**다루는 내용**:
- Mutex 비교 (CRITICAL_SECTION vs pthread_mutex_t)
- Condition Variable (Event vs pthread_cond_t)
- Semaphore (Windows Semaphore vs POSIX sem_t)
- Reader-Writer Lock (SRWLock vs pthread_rwlock_t)
- 성능 차이 분석

**핵심 차이**:
| 동기화 객체 | Windows | POSIX |
|------------|---------|-------|
| **경량 Mutex** | CRITICAL_SECTION | pthread_mutex_t |
| **프로세스 간 Mutex** | Mutex Object | Named Semaphore |
| **Condition Variable** | Condition Variable (Vista+) | pthread_cond_t |
| **수동 리셋 이벤트** | Event (Manual Reset) | 없음 (조합으로 구현) |
| **RWLock** | SRWLock (Vista+) | pthread_rwlock_t |

---

### [03. Thread Local Storage (TLS)](./03-thread-local-storage.md)
**핵심 비교**: TlsAlloc vs pthread_key_create

**다루는 내용**:
- TLS API 비교
- 정적 TLS vs 동적 TLS
- __declspec(thread) vs __thread
- TLS 초기화 및 정리
- 성능 고려사항

**예시**:
```cpp
// Windows
DWORD tlsIndex = TlsAlloc();
TlsSetValue(tlsIndex, pData);
void* data = TlsGetValue(tlsIndex);

// POSIX
pthread_key_t key;
pthread_key_create(&key, destructor);
pthread_setspecific(key, pData);
void* data = pthread_getspecific(key);
```

---

### [04. 프로세스 간 통신 (IPC)](./04-ipc.md)
**핵심 비교**: Named Objects vs POSIX IPC

**다루는 내용**:
- Named Mutex/Semaphore/Event
- Shared Memory (File Mapping vs shm_open)
- Named Pipes vs Unix Domain Sockets
- Message Queues
- 성능 및 보안 비교

---

### [05. 스케줄링 및 우선순위](./05-scheduling.md)
**핵심 비교**: Priority Classes vs Nice Values

**다루는 내용**:
- Windows Priority Classes (Real-time, High, Normal, Idle)
- POSIX Scheduling Policies (SCHED_FIFO, SCHED_RR, SCHED_OTHER)
- Thread Affinity (SetThreadAffinityMask vs pthread_setaffinity_np)
- Processor Groups (Windows)
- Real-Time Scheduling

---

### [06. 에러 처리](./06-error-handling.md)
**핵심 비교**: GetLastError vs errno

**다루는 내용**:
- 에러 코드 체계
- 스레드 안전 에러 처리
- 에러 메시지 포매팅
- 디버깅 및 로깅

---

### [07. 이식성 레이어](./07-portability-layer.md)
**핵심**: 크로스 플랫폼 추상화

**다루는 내용**:
- 추상화 레이어 설계
- C++11 std::thread 사용 (권장)
- Boost.Thread
- 조건부 컴파일 (#ifdef _WIN32)
- CMake를 이용한 플랫폼 감지

**예시**:
```cpp
// 이식 가능한 추상화
#ifdef _WIN32
    #include <windows.h>
    typedef HANDLE thread_t;
#else
    #include <pthread.h>
    typedef pthread_t thread_t;
#endif

// 또는 C++11 표준 사용 (권장)
#include <thread>
std::thread t(func);
```

---

### [08. 성능 비교](./08-performance-comparison.md)
**핵심**: 벤치마크 및 최적화

**다루는 내용**:
- 스레드 생성 오버헤드
- 컨텍스트 스위칭 속도
- 동기화 프리미티브 성능
- 메모리 모델 차이
- 실측 벤치마크 결과

---

## 🎓 플랫폼별 철학

### Windows 철학

**"Everything is an object"**
- 모든 리소스가 커널 객체 (핸들)
- 일관된 Wait API (WaitForSingleObject, WaitForMultipleObjects)
- GUI 통합 (Message Queue, Window Messages)

**특징**:
- ✅ 풍부한 동기화 객체 (Event, Semaphore, Mutex 모두 별도 제공)
- ✅ WaitForMultipleObjects로 여러 객체 동시 대기
- ⚠️ 핸들 관리 복잡성
- ⚠️ Windows 전용 (이식성 낮음)

### POSIX 철학

**"Simple is better"**
- 최소한의 프리미티브 제공
- Condition Variable + Mutex 조합으로 대부분 해결
- 파일 디스크립터 중심

**특징**:
- ✅ 간결한 API
- ✅ 높은 이식성 (Linux, macOS, BSD, Unix)
- ✅ 표준화된 인터페이스
- ⚠️ Event 같은 고수준 추상화 없음 (직접 구현 필요)

---

## 📊 API 매핑 표

### 스레드 생성

| 기능 | Windows | POSIX |
|------|---------|-------|
| **생성** | CreateThread | pthread_create |
| **종료 대기** | WaitForSingleObject | pthread_join |
| **분리** | CloseHandle (즉시) | pthread_detach |
| **종료** | ExitThread | pthread_exit |
| **강제 종료** | TerminateThread (위험) | pthread_cancel |
| **ID 얻기** | GetCurrentThreadId | pthread_self |

### 동기화

| 기능 | Windows | POSIX |
|------|---------|-------|
| **Mutex (경량)** | CRITICAL_SECTION | pthread_mutex_t |
| **Mutex (프로세스 간)** | CreateMutex | Named Semaphore |
| **Condition Variable** | CONDITION_VARIABLE | pthread_cond_t |
| **Semaphore** | CreateSemaphore | sem_open/sem_init |
| **RWLock** | SRWLock | pthread_rwlock_t |
| **Event (Auto Reset)** | CreateEvent (Auto) | CV + Mutex |
| **Event (Manual Reset)** | CreateEvent (Manual) | CV + Flag + Mutex |
| **여러 객체 대기** | WaitForMultipleObjects | select/poll/epoll |

### TLS

| 기능 | Windows | POSIX |
|------|---------|-------|
| **동적 TLS** | TlsAlloc/TlsSetValue | pthread_key_create |
| **정적 TLS** | __declspec(thread) | __thread (GCC) |

---

## 💡 크로스 플랫폼 개발 권장사항

### 1. C++11 표준 사용 (최우선 권장)

```cpp
#include <thread>
#include <mutex>
#include <condition_variable>

std::thread t(func);
std::mutex mtx;
std::condition_variable cv;
```

**장점**:
- ✅ 완전한 크로스 플랫폼
- ✅ RAII 지원
- ✅ 타입 안전
- ✅ 최신 기능 (move semantics, lambda)

### 2. 플랫폼 추상화 레이어

필요한 경우 (C++11 이전, 특수 기능):

```cpp
class Thread {
public:
    #ifdef _WIN32
        HANDLE handle;
    #else
        pthread_t handle;
    #endif

    void start(void* (*func)(void*), void* arg);
    void join();
};
```

### 3. 조건부 컴파일

```cpp
#ifdef _WIN32
    #include <windows.h>
    #define SLEEP_MS(ms) Sleep(ms)
#else
    #include <unistd.h>
    #define SLEEP_MS(ms) usleep((ms) * 1000)
#endif
```

### 4. 빌드 시스템 (CMake)

```cmake
if(WIN32)
    target_compile_definitions(myapp PRIVATE PLATFORM_WINDOWS)
elseif(UNIX)
    target_compile_definitions(myapp PRIVATE PLATFORM_POSIX)
    target_link_libraries(myapp pthread)
endif()
```

---

## ⚠️ 주의사항

### Windows → POSIX 포팅

1. **WaitForMultipleObjects 대체**
   - POSIX에는 없음
   - select/poll/epoll (파일 디스크립터)
   - 여러 CV를 별도 스레드로 모니터링

2. **Event 구현**
   - Manual Reset Event: CV + bool flag
   - Auto Reset Event: CV + counter

3. **핸들 정리**
   - Windows: CloseHandle 필수
   - POSIX: pthread_join 또는 pthread_detach

### POSIX → Windows 포팅

1. **pthread_cancel 대체**
   - Windows에 직접 대응 없음
   - 협력적 취소 (플래그) 권장

2. **fork() 없음**
   - Windows는 CreateProcess 사용
   - 멀티프로세스 아키텍처 재설계 필요

3. **Signals 차이**
   - POSIX signals는 Windows에서 제한적
   - 대안: Event, Message Queue

---

## 🔗 다음 단계

- [스레드 생성 및 관리](./01-thread-creation.md) - API 상세 비교
- [동기화 프리미티브](./02-synchronization-primitives.md) - Mutex, CV 등
- [이식성 레이어](./07-portability-layer.md) - 크로스 플랫폼 추상화

---

## 📚 참고 자료

### 공식 문서
- [Microsoft Win32 Threading](https://docs.microsoft.com/en-us/windows/win32/procthread/processes-and-threads)
- [POSIX Threads Programming](https://computing.llnl.gov/tutorials/pthreads/)
- [The Open Group POSIX Specification](https://pubs.opengroup.org/onlinepubs/9699919799/)

### 서적
- "Windows System Programming" - Johnson M. Hart
- "Programming with POSIX Threads" - David R. Butenhof
- "Windows Internals" - Mark Russinovich

---

*크로스 플랫폼 멀티스레드 프로그래밍은 어렵지만, 표준(C++11)을 사용하거나 올바른 추상화로 관리 가능합니다!*
