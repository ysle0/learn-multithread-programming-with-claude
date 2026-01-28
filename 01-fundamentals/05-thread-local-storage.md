# Thread Local Storage (TLS) 심층 분석

## 목차
1. [개요](#개요)
2. [TLS가 필요한 이유](#tls가-필요한-이유)
3. [내부 동작 원리](#내부-동작-원리)
4. [정적 TLS vs 동적 TLS](#정적-tls-vs-동적-tls)
5. [ELF TLS 모델](#elf-tls-모델)
6. [Use Cases](#use-cases)
7. [성능 분석](#성능-분석)
8. [주의사항 및 Best Practices](#주의사항-및-best-practices)
9. [요약](#요약)

---

## 개요

**Thread Local Storage (TLS)**는 각 스레드가 고유한 데이터 복사본을 가질 수 있게 하는 메커니즘입니다. 전역 변수처럼 보이지만, 실제로는 각 스레드에서 독립적인 값을 가집니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Process Memory                           │
├─────────────────────────────────────────────────────────────────┤
│  Global Variables (Shared)                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  int shared_counter = 0;  ← 모든 스레드가 공유           │   │
│  └─────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  Thread Local Storage (Per-Thread)                              │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐       │
│  │   Thread 1    │  │   Thread 2    │  │   Thread 3    │       │
│  │ ┌───────────┐ │  │ ┌───────────┐ │  │ ┌───────────┐ │       │
│  │ │ errno = 0 │ │  │ │ errno = 5 │ │  │ │ errno = 2 │ │       │
│  │ │ cache[..] │ │  │ │ cache[..] │ │  │ │ cache[..] │ │       │
│  │ │ pool_ptr  │ │  │ │ pool_ptr  │ │  │ │ pool_ptr  │ │       │
│  │ └───────────┘ │  │ └───────────┘ │  │ └───────────┘ │       │
│  └───────────────┘  └───────────────┘  └───────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### 핵심 특성

| 특성 | 전역 변수 | TLS 변수 |
|------|----------|---------|
| 메모리 공간 | 프로세스당 1개 | 스레드당 1개 |
| 동기화 필요 | 필수 (Lock 등) | 불필요 |
| 접근 성능 | 가장 빠름 | 약간의 오버헤드 |
| 초기화 | 프로세스 시작 시 | 스레드 생성 시 |

---

## TLS가 필요한 이유

### 1. 동기화 없는 스레드 안전성

가장 큰 이유는 **락 없이 스레드 안전한 코드**를 작성할 수 있다는 것입니다.

```c
// 문제: 전역 변수는 Race Condition 발생
int global_counter = 0;

void increment() {
    global_counter++;  // Race Condition!
}

// 해결책 1: 락 사용 (성능 저하)
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
void safe_increment() {
    pthread_mutex_lock(&mutex);
    global_counter++;
    pthread_mutex_unlock(&mutex);
}

// 해결책 2: TLS 사용 (락 불필요)
__thread int thread_counter = 0;

void tls_increment() {
    thread_counter++;  // 스레드별 독립 → 동기화 불필요
}
```

### 2. errno 문제

TLS의 대표적인 사용 사례는 `errno`입니다.

```c
// 싱글스레드 시대의 errno
extern int errno;  // 전역 변수

// 문제: 멀티스레드에서 다른 스레드가 errno를 덮어씀
Thread 1: open("/nonexistent", O_RDONLY);  // errno = ENOENT
Thread 2: write(invalid_fd, buf, len);     // errno = EBADF ← Thread 1의 errno 덮어씀
Thread 1: if (errno == ENOENT) { ... }     // 잘못된 errno 값!

// 해결: TLS 기반 errno
#define errno (*__errno_location())  // 스레드별 errno 반환

// 각 스레드가 독립적인 errno를 가짐
Thread 1: errno = ENOENT (Thread 1 전용)
Thread 2: errno = EBADF  (Thread 2 전용)
```

### 3. 컨텍스트 전달의 어려움

깊은 함수 호출 체인에서 컨텍스트를 전달하기 어려울 때 TLS가 유용합니다.

```c
// 문제: 모든 함수에 context를 전달해야 함
void process_request(Context* ctx, Request* req) {
    validate_request(ctx, req);
    // ctx를 계속 전달해야 함...
}

void validate_request(Context* ctx, Request* req) {
    log_message(ctx, "Validating...");
    // 또 ctx 전달...
}

void log_message(Context* ctx, const char* msg) {
    printf("[%s] %s\n", ctx->request_id, msg);
}

// 해결: TLS로 컨텍스트 저장
__thread Context* current_context = NULL;

void set_context(Context* ctx) {
    current_context = ctx;
}

Context* get_context() {
    return current_context;
}

// 이제 함수 시그니처가 깔끔해짐
void log_message(const char* msg) {
    Context* ctx = get_context();
    printf("[%s] %s\n", ctx->request_id, msg);
}
```

---

## 내부 동작 원리

TLS는 단순해 보이지만, 효율적인 구현을 위해 **컴파일러, 링커, 동적 링커, 커널, 런타임**이 협력해야 합니다.

### x86-64 아키텍처: FS/GS 세그먼트 레지스터

x86-64에서 TLS는 **FS 세그먼트 레지스터**를 통해 구현됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                     x86-64 CPU Registers                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  General Purpose: RAX, RBX, RCX, RDX, RSI, RDI, ...            │
│                                                                 │
│  Segment Registers:                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ CS, SS, DS, ES → 64비트 모드에서 무시됨 (base = 0)       │   │
│  │                                                          │   │
│  │ FS → Thread Local Storage (User-space TLS)              │   │
│  │ GS → Kernel per-CPU data (Linux)                        │   │
│  │      또는 TEB (Windows x64)                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Model Specific Registers (MSR):                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ MSR_FS_BASE (0xC0000100) → FS의 base address            │   │
│  │ MSR_GS_BASE (0xC0000101) → GS의 base address            │   │
│  │ MSR_KERNEL_GS_BASE (0xC0000102) → SWAPGS용              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### TLS 변수 접근 과정

```nasm
; TLS 변수 접근 (x86-64)
; __thread int my_var = 42;
; int x = my_var;

mov eax, dword ptr fs:[0xfffffffc]  ; FS base + offset(-4)
                                     ; offset은 컴파일 타임에 결정됨
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    TLS Access (x86-64)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  FS Register                                                    │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────┐                                                │
│  │ MSR_FS_BASE │───────────────────────────┐                   │
│  └─────────────┘                            │                   │
│                                             ▼                   │
│                              ┌──────────────────────────┐       │
│                              │   Thread Control Block   │       │
│                              │         (TCB)            │       │
│                              ├──────────────────────────┤       │
│   Negative Offset            │ ... padding ...          │       │
│   (컴파일 타임 결정)          │                          │       │
│           │                  ├──────────────────────────┤ ← -4  │
│           └─────────────────►│ my_var = 42              │       │
│                              ├──────────────────────────┤ ← 0   │
│                              │ self pointer             │       │
│                              │ dtv pointer              │       │
│                              │ thread ID                │       │
│                              │ ...                      │       │
│                              └──────────────────────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Thread Control Block (TCB)

각 스레드는 **TCB (Thread Control Block)**라는 메타데이터 구조체를 가집니다.

```c
// Linux glibc의 TCB (struct pthread 일부)
struct pthread {
    // TLS 관련
    void *self;                    // TCB 자신을 가리키는 포인터
    dtv_t *dtv;                    // Dynamic Thread Vector

    // 스레드 메타데이터
    pid_t tid;                     // Thread ID
    pthread_t pthread_id;          // pthread handle

    // 동기화 관련
    int cancelhandling;
    struct pthread_mutex *robust_list;

    // ... 더 많은 필드
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│                 Memory Layout per Thread                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Low Address                                                    │
│       │                                                         │
│       ▼                                                         │
│  ┌──────────────────────┐                                       │
│  │                      │                                       │
│  │    Thread Stack      │                                       │
│  │                      │                                       │
│  ├──────────────────────┤                                       │
│  │    Guard Page        │  ← Stack overflow 방지                │
│  ├──────────────────────┤                                       │
│  │                      │                                       │
│  │   Static TLS Block   │  ← __thread 변수들                    │
│  │   (.tdata + .tbss)   │                                       │
│  │                      │                                       │
│  ├──────────────────────┤ ← FS register가 가리키는 위치         │
│  │                      │                                       │
│  │   TCB (pthread)      │  ← Thread Control Block               │
│  │                      │                                       │
│  └──────────────────────┘                                       │
│       │                                                         │
│       ▼                                                         │
│  High Address                                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Dynamic Thread Vector (DTV)

동적으로 로드된 공유 라이브러리의 TLS 변수를 위해 **DTV**가 사용됩니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                 Dynamic Thread Vector (DTV)                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TCB                                                            │
│  ┌────────────┐                                                 │
│  │ dtv ───────┼─────────┐                                       │
│  └────────────┘         │                                       │
│                         ▼                                       │
│            ┌────────────────────────────┐                       │
│            │ DTV (Dynamic Thread Vector)│                       │
│            ├────────────────────────────┤                       │
│  index 0   │ generation counter         │                       │
│            ├────────────────────────────┤                       │
│  index 1   │ main executable TLS ───────┼───► TLS Block 1       │
│            ├────────────────────────────┤                       │
│  index 2   │ libfoo.so TLS ─────────────┼───► TLS Block 2       │
│            ├────────────────────────────┤                       │
│  index 3   │ libbar.so TLS ─────────────┼───► TLS Block 3       │
│            ├────────────────────────────┤     (lazy allocation) │
│  index 4   │ NULL (not loaded yet)      │                       │
│            └────────────────────────────┘                       │
│                                                                 │
│  TLS 접근: dtv[module_id].pointer + offset                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 컨텍스트 스위칭과 TLS

스레드 전환 시 커널은 FS base 레지스터를 교체합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Context Switch & TLS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread A Running                    Thread B Running           │
│  ┌─────────────┐                    ┌─────────────┐             │
│  │ FS → TCB_A  │                    │ FS → TCB_B  │             │
│  └─────────────┘                    └─────────────┘             │
│        │                                  ▲                     │
│        │    Context Switch                │                     │
│        │    ════════════════════════════► │                     │
│        │                                  │                     │
│        │    1. Save Thread A state        │                     │
│        │    2. Save FS base (MSR_FS_BASE) │                     │
│        │    3. Load Thread B's FS base    │                     │
│        │    4. Restore Thread B state     │                     │
│        │                                  │                     │
│                                                                 │
│  Kernel switch_to() (Linux):                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  // TLS 로드 (컨텍스트 스위칭의 일부)                      │   │
│  │  load_TLS(next_thread);                                  │   │
│  │                                                          │   │
│  │  // MSR_FS_BASE 업데이트                                  │   │
│  │  wrmsrl(MSR_FS_BASE, next_thread->fsbase);               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 정적 TLS vs 동적 TLS

### 정적 TLS (Implicit TLS)

컴파일러 키워드로 선언하는 방식입니다.

```c
// POSIX (GCC/Clang)
__thread int counter = 0;

// Windows (MSVC)
__declspec(thread) int counter = 0;

// C11 표준
_Thread_local int counter = 0;

// C++11 표준
thread_local int counter = 0;
```

**장점:**
- 빠른 접근 (몇 개의 명령어)
- 간단한 문법
- 자동 초기화/정리

**단점:**
- 동적 라이브러리(dlopen)에서 제한
- 컴파일 타임에 결정
- 메모리가 모든 스레드에 할당됨

### 동적 TLS (Explicit TLS)

런타임 API로 할당하는 방식입니다.

```c
// POSIX
pthread_key_t key;
pthread_key_create(&key, destructor);
pthread_setspecific(key, value);
value = pthread_getspecific(key);
pthread_key_delete(key);

// Windows
DWORD index = TlsAlloc();
TlsSetValue(index, value);
value = TlsGetValue(index);
TlsFree(index);
```

**장점:**
- 동적 라이브러리에서 안전
- 필요한 스레드만 할당
- 정리 함수(destructor) 지원 (POSIX)

**단점:**
- 함수 호출 오버헤드
- 명시적 초기화/정리 필요
- 키 개수 제한 (PTHREAD_KEYS_MAX)

### 비교

| 항목 | 정적 TLS | 동적 TLS |
|------|---------|---------|
| 선언 | `__thread int x;` | `pthread_key_t key;` |
| 접근 속도 | ~1-2 ns | ~5-20 ns |
| 메모리 | 모든 스레드에 할당 | 필요 시 할당 |
| 초기화 | 자동 (0 또는 지정값) | 수동 |
| 정리 | 자동 | 수동 (destructor 가능) |
| dlopen | 제한적 | 안전 |

---

## ELF TLS 모델

컴파일러와 링커는 TLS 접근 방식을 **4가지 모델**로 구분합니다.

### 1. Local Exec (LE) - 가장 빠름

실행 파일 내 TLS 변수에 직접 접근합니다.

```nasm
; 가장 효율적 - 단일 명령어
mov eax, dword ptr fs:[tls_var@tpoff]
```

- **사용 조건**: 실행 파일 내 정의된 TLS 변수
- **특징**: 컴파일 타임에 offset 결정

### 2. Initial Exec (IE)

프로그램 시작 시 로드된 공유 라이브러리의 TLS입니다.

```nasm
; GOT를 통한 간접 접근
mov rax, qword ptr [rip + tls_var@gottpoff]
mov eax, dword ptr fs:[rax]
```

- **사용 조건**: 정적 링크된 공유 라이브러리
- **특징**: GOT 참조 필요

### 3. Local Dynamic (LD)

같은 모듈 내 여러 TLS 변수 접근 최적화입니다.

```nasm
; __tls_get_addr 호출 1회로 모듈 base 획득
lea rdi, [rip + _TLS_MODULE_BASE_@tlsld]
call __tls_get_addr
; 이후 offset만으로 접근
mov ecx, dword ptr [rax + tls_var1@dtpoff]
mov edx, dword ptr [rax + tls_var2@dtpoff]
```

- **사용 조건**: -fpic로 컴파일된 공유 라이브러리
- **특징**: 같은 모듈 내 여러 변수 접근 시 효율적

### 4. General Dynamic (GD) - 가장 느림

가장 일반적이지만 가장 느린 방식입니다.

```nasm
; 매 접근마다 __tls_get_addr 호출
lea rdi, [rip + tls_var@tlsgd]
call __tls_get_addr
mov eax, dword ptr [rax]
```

- **사용 조건**: dlopen으로 로드된 라이브러리
- **특징**: 완전한 유연성, 최대 오버헤드

### TLS 모델 선택 (컴파일러 옵션)

```bash
# GCC/Clang
gcc -ftls-model=local-exec    # 실행 파일용
gcc -ftls-model=initial-exec  # 정적 링크 라이브러리
gcc -ftls-model=local-dynamic # 공유 라이브러리 (같은 모듈 내)
gcc -ftls-model=global-dynamic # 기본값 (가장 안전)
```

### 링커의 TLS Relaxation

링커는 더 효율적인 모델로 "완화(relax)"할 수 있습니다.

```
GD → LD  (같은 모듈 내 여러 변수)
GD → IE  (프로그램 시작 시 로드)
GD → LE  (실행 파일로 링크)
IE → LE  (실행 파일로 링크)
```

---

## Use Cases

### 1. Thread-Local Memory Pool

**가장 중요한 use case 중 하나**입니다. 각 스레드가 독립적인 메모리 풀을 가지면 할당/해제 시 락이 필요 없습니다.

```c
#include <pthread.h>
#include <stdlib.h>
#include <stdint.h>

#define BLOCK_SIZE 64
#define POOL_CAPACITY 1024

// 스레드별 메모리 풀 구조체
typedef struct {
    void* blocks[POOL_CAPACITY];
    size_t count;
    size_t allocated_total;
    size_t freed_total;
} ThreadLocalPool;

// TLS로 선언된 스레드별 풀
__thread ThreadLocalPool* tl_pool = NULL;

// 풀 초기화
void init_thread_pool() {
    if (tl_pool == NULL) {
        tl_pool = (ThreadLocalPool*)malloc(sizeof(ThreadLocalPool));
        tl_pool->count = 0;
        tl_pool->allocated_total = 0;
        tl_pool->freed_total = 0;
    }
}

// 락 없는 할당
void* pool_alloc() {
    init_thread_pool();

    if (tl_pool->count > 0) {
        // 캐시된 블록 반환 (락 불필요!)
        tl_pool->count--;
        return tl_pool->blocks[tl_pool->count];
    }

    // 캐시 비어있으면 시스템 할당
    tl_pool->allocated_total++;
    return malloc(BLOCK_SIZE);
}

// 락 없는 해제
void pool_free(void* ptr) {
    if (ptr == NULL) return;
    init_thread_pool();

    if (tl_pool->count < POOL_CAPACITY) {
        // 캐시에 저장 (락 불필요!)
        tl_pool->blocks[tl_pool->count++] = ptr;
        return;
    }

    // 캐시 가득 차면 시스템 해제
    tl_pool->freed_total++;
    free(ptr);
}

// 스레드 종료 시 정리
void cleanup_thread_pool() {
    if (tl_pool != NULL) {
        for (size_t i = 0; i < tl_pool->count; i++) {
            free(tl_pool->blocks[i]);
        }
        free(tl_pool);
        tl_pool = NULL;
    }
}
```

#### TCMalloc의 Thread Cache

Google의 TCMalloc은 이 패턴의 대표적인 실제 구현입니다.

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCMalloc Architecture                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  Thread 1   │  │  Thread 2   │  │  Thread 3   │             │
│  │             │  │             │  │             │             │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │             │
│  │ │ Thread  │ │  │ │ Thread  │ │  │ │ Thread  │ │             │
│  │ │ Cache   │ │  │ │ Cache   │ │  │ │ Cache   │ │ ← TLS       │
│  │ │ (TLS)   │ │  │ │ (TLS)   │ │  │ │ (TLS)   │ │             │
│  │ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │             │
│  └──────┼──────┘  └──────┼──────┘  └──────┼──────┘             │
│         │                │                │                     │
│         │   Overflow/    │                │                     │
│         │   Underflow    │                │                     │
│         ▼                ▼                ▼                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Central Heap                          │   │
│  │                 (Lock Required)                          │   │
│  │  ┌───────────────────────────────────────────────────┐  │   │
│  │  │ Size Class 0 │ Size Class 1 │ ... │ Size Class N  │  │   │
│  │  └───────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      Page Heap                           │   │
│  │              (Large Object / Span 관리)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  할당 경로:                                                     │
│  1. Thread Cache에서 할당 시도 (락 없음!)                       │
│  2. Thread Cache 비어있으면 → Central Heap에서 가져옴           │
│  3. Central Heap 비어있으면 → Page Heap에서 할당                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2. errno (스레드별 에러 코드)

표준 C 라이브러리의 `errno`는 TLS로 구현됩니다.

```c
// glibc 구현 (단순화)
// errno.h
extern int *__errno_location(void);
#define errno (*__errno_location())

// 실제 구현
__thread int __libc_errno = 0;

int *__errno_location(void) {
    return &__libc_errno;
}

// 사용 예
void safe_read(int fd, void* buf, size_t count) {
    ssize_t result = read(fd, buf, count);
    if (result < 0) {
        // 각 스레드는 독립적인 errno를 가짐
        switch (errno) {  // 다른 스레드에 의해 덮어써지지 않음
            case EAGAIN:
                // retry...
                break;
            case EINTR:
                // interrupted...
                break;
        }
    }
}
```

### 3. Thread-Local Cache

자주 접근하는 데이터를 스레드별로 캐싱합니다.

```c
#include <pthread.h>
#include <time.h>
#include <string.h>

// 스레드별 시간 캐시
typedef struct {
    time_t cached_time;
    char formatted[64];
    int valid;
} TimeCache;

__thread TimeCache time_cache = {0, "", 0};

// 락 없이 포맷된 시간 문자열 반환
const char* get_formatted_time() {
    time_t now = time(NULL);

    // 캐시 유효성 검사 (1초 이내면 캐시 사용)
    if (time_cache.valid && (now - time_cache.cached_time) < 1) {
        return time_cache.formatted;
    }

    // 캐시 갱신 (각 스레드 독립적)
    struct tm* tm_info = localtime(&now);
    strftime(time_cache.formatted, sizeof(time_cache.formatted),
             "%Y-%m-%d %H:%M:%S", tm_info);
    time_cache.cached_time = now;
    time_cache.valid = 1;

    return time_cache.formatted;
}
```

### 4. Thread-Local Random Generator

스레드별 난수 생성기 상태를 유지합니다.

```c
#include <stdint.h>

// 스레드별 난수 생성기 상태
typedef struct {
    uint64_t state;
    uint64_t inc;
} PCG32State;

__thread PCG32State rng_state = {0x853c49e6748fea9bULL, 0xda3e39cb94b95bdbULL};

// 스레드별 시드 설정
void seed_thread_rng(uint64_t seed) {
    rng_state.state = seed;
    rng_state.inc = (seed << 1) | 1;
}

// 락 없는 난수 생성 (PCG32 알고리즘)
uint32_t thread_random() {
    uint64_t old_state = rng_state.state;
    rng_state.state = old_state * 6364136223846793005ULL + rng_state.inc;

    uint32_t xorshifted = ((old_state >> 18u) ^ old_state) >> 27u;
    uint32_t rot = old_state >> 59u;
    return (xorshifted >> rot) | (xorshifted << ((-rot) & 31));
}

// 사용 예
void* worker_thread(void* arg) {
    // 스레드 ID로 시드 설정
    seed_thread_rng((uint64_t)pthread_self());

    for (int i = 0; i < 1000; i++) {
        uint32_t r = thread_random();  // 락 불필요
        // ... use random number ...
    }
    return NULL;
}
```

### 5. Request Context (웹 서버)

웹 서버에서 요청별 컨텍스트를 관리합니다.

```c
#include <pthread.h>
#include <uuid/uuid.h>

typedef struct {
    char request_id[37];
    char user_id[64];
    time_t start_time;
    int log_level;
} RequestContext;

__thread RequestContext* current_request = NULL;

void begin_request(const char* user_id) {
    current_request = malloc(sizeof(RequestContext));

    uuid_t uuid;
    uuid_generate(uuid);
    uuid_unparse(uuid, current_request->request_id);

    strncpy(current_request->user_id, user_id, sizeof(current_request->user_id));
    current_request->start_time = time(NULL);
    current_request->log_level = LOG_INFO;
}

void end_request() {
    free(current_request);
    current_request = NULL;
}

// 어디서든 현재 요청 컨텍스트 접근 가능
void log_message(int level, const char* format, ...) {
    if (current_request == NULL || level < current_request->log_level) {
        return;
    }

    // 자동으로 request_id 포함
    printf("[%s] [%s] ", current_request->request_id, current_request->user_id);

    va_list args;
    va_start(args, format);
    vprintf(format, args);
    va_end(args);
    printf("\n");
}
```

### 6. Cross-Thread Deallocation 처리

한 스레드에서 할당하고 다른 스레드에서 해제하는 경우입니다.

```c
#include <pthread.h>
#include <stdatomic.h>

typedef struct FreeNode {
    struct FreeNode* next;
} FreeNode;

typedef struct {
    // 스레드 자신의 free list (락 불필요)
    FreeNode* private_list;

    // 다른 스레드가 반환한 블록 (lock-free)
    _Atomic(FreeNode*) public_list;
} ThreadLocalAllocator;

__thread ThreadLocalAllocator* tl_allocator = NULL;

// 같은 스레드에서 해제 → private list (빠름)
void local_free(void* ptr) {
    FreeNode* node = (FreeNode*)ptr;
    node->next = tl_allocator->private_list;
    tl_allocator->private_list = node;
}

// 다른 스레드에서 해제 → public list (lock-free CAS)
void remote_free(ThreadLocalAllocator* target, void* ptr) {
    FreeNode* node = (FreeNode*)ptr;
    FreeNode* old_head;
    do {
        old_head = atomic_load(&target->public_list);
        node->next = old_head;
    } while (!atomic_compare_exchange_weak(&target->public_list, &old_head, node));
}

// 할당 시 public list를 private list로 병합
void* local_alloc(size_t size) {
    // private list가 비어있으면 public list 병합
    if (tl_allocator->private_list == NULL) {
        FreeNode* public_head = atomic_exchange(&tl_allocator->public_list, NULL);
        tl_allocator->private_list = public_head;
    }

    if (tl_allocator->private_list != NULL) {
        FreeNode* node = tl_allocator->private_list;
        tl_allocator->private_list = node->next;
        return node;
    }

    return malloc(size);
}
```

---

## 성능 분석

### TLS 접근 비용 비교

```
┌─────────────────────────────────────────────────────────────────┐
│                    TLS Access Cost Comparison                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Access Method                    Approximate Cost (cycles)     │
│  ─────────────────────────────────────────────────────────      │
│                                                                 │
│  Register                         ~0.3 cycles                   │
│  ▓                                                              │
│                                                                 │
│  Local Variable (stack)           ~0.5 cycles                   │
│  ▓▓                                                             │
│                                                                 │
│  Global Variable                  ~1-2 cycles                   │
│  ▓▓▓▓                                                           │
│                                                                 │
│  Static TLS (Local Exec)          ~1-3 cycles                   │
│  ▓▓▓▓▓                                                          │
│                                                                 │
│  Static TLS (Initial Exec)        ~3-5 cycles                   │
│  ▓▓▓▓▓▓▓▓▓                                                      │
│                                                                 │
│  Dynamic TLS (pthread_getspecific) ~10-30 cycles               │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓                                      │
│                                                                 │
│  General Dynamic TLS              ~50-100 cycles               │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓           │
│                                                                 │
│  Mutex Lock + Access + Unlock     ~100-1000+ cycles            │
│  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓... │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 벤치마크 코드

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <time.h>

#define ITERATIONS 100000000

// 전역 변수
int global_var = 0;

// 정적 TLS
__thread int static_tls_var = 0;

// 동적 TLS
pthread_key_t dynamic_tls_key;

static inline long long get_ns() {
    struct timespec ts;
    clock_gettime(CLOCK_MONOTONIC, &ts);
    return ts.tv_sec * 1000000000LL + ts.tv_nsec;
}

void benchmark_global() {
    long long start = get_ns();
    volatile int sum = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        sum += global_var;
        global_var = i;
    }
    long long end = get_ns();
    printf("Global Variable: %.2f ns/access\n",
           (double)(end - start) / ITERATIONS);
}

void benchmark_static_tls() {
    long long start = get_ns();
    volatile int sum = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        sum += static_tls_var;
        static_tls_var = i;
    }
    long long end = get_ns();
    printf("Static TLS (__thread): %.2f ns/access\n",
           (double)(end - start) / ITERATIONS);
}

void benchmark_dynamic_tls() {
    pthread_setspecific(dynamic_tls_key, (void*)0);

    long long start = get_ns();
    volatile intptr_t sum = 0;
    for (int i = 0; i < ITERATIONS; i++) {
        sum += (intptr_t)pthread_getspecific(dynamic_tls_key);
        pthread_setspecific(dynamic_tls_key, (void*)(intptr_t)i);
    }
    long long end = get_ns();
    printf("Dynamic TLS (pthread): %.2f ns/access\n",
           (double)(end - start) / ITERATIONS);
}

int main() {
    pthread_key_create(&dynamic_tls_key, NULL);

    printf("=== TLS Performance Benchmark ===\n");
    printf("Iterations: %d\n\n", ITERATIONS);

    benchmark_global();
    benchmark_static_tls();
    benchmark_dynamic_tls();

    pthread_key_delete(dynamic_tls_key);
    return 0;
}
```

### 일반적인 결과

| 방식 | 접근 시간 | 사용 권장 상황 |
|------|----------|---------------|
| 정적 TLS (LE) | ~0.5-2 ns | 실행 파일 내 빈번한 접근 |
| 정적 TLS (IE) | ~2-5 ns | 공유 라이브러리 내 빈번한 접근 |
| 동적 TLS | ~10-30 ns | 라이브러리, destructor 필요 시 |
| Mutex + Global | ~100-1000 ns | 경합이 적은 경우 |

---

## 주의사항 및 Best Practices

### 1. 메모리 누수 방지

동적 TLS는 반드시 destructor를 등록하거나 수동 정리해야 합니다.

```c
// 잘못된 예: 메모리 누수
pthread_key_t key;
pthread_key_create(&key, NULL);  // destructor 없음!

void* thread_func(void* arg) {
    char* data = malloc(1024);
    pthread_setspecific(key, data);
    // 스레드 종료 시 data가 해제되지 않음!
    return NULL;
}

// 올바른 예: destructor 등록
void cleanup(void* data) {
    free(data);
}

pthread_key_create(&key, cleanup);  // 자동 정리
```

### 2. 스레드 풀에서의 주의

스레드 풀은 스레드를 재사용하므로 TLS 값이 유지됩니다.

```c
// 문제: 이전 작업의 TLS 값이 남아있음
__thread RequestContext* ctx = NULL;

void handle_request() {
    // ctx가 이전 요청의 값을 가지고 있을 수 있음!
    if (ctx != NULL) {
        // 잘못된 컨텍스트 사용 가능
    }

    ctx = create_context();
    // ... 처리 ...
}

// 해결: 작업 시작/종료 시 명시적 초기화/정리
void handle_request() {
    // 시작 시 초기화
    ctx = create_context();

    // ... 처리 ...

    // 종료 시 정리
    destroy_context(ctx);
    ctx = NULL;
}
```

### 3. DLL/공유 라이브러리 제한

Windows에서 정적 TLS는 LoadLibrary로 로드된 DLL에서 문제가 됩니다.

```c
// Windows DLL에서 정적 TLS 사용 시 주의
// Vista 이전: LoadLibrary로 로드된 DLL에서 동작 안 함
// Vista 이후: 동작하지만 오버헤드 있음

// 해결: 동적 TLS 사용
DWORD g_tlsIndex;

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    switch (fdwReason) {
        case DLL_PROCESS_ATTACH:
            g_tlsIndex = TlsAlloc();
            break;
        case DLL_THREAD_DETACH:
            // 정리
            TlsFree(TlsGetValue(g_tlsIndex));
            break;
        case DLL_PROCESS_DETACH:
            TlsFree(g_tlsIndex);
            break;
    }
    return TRUE;
}
```

### 4. TLS 값 캐싱

루프 내에서 TLS를 반복 접근하면 로컬 변수에 캐시하세요.

```c
// 비효율적
for (int i = 0; i < 1000000; i++) {
    tls_counter++;  // 매번 TLS 접근
}

// 효율적
int local_counter = tls_counter;  // 한 번만 읽기
for (int i = 0; i < 1000000; i++) {
    local_counter++;
}
tls_counter = local_counter;  // 한 번만 쓰기
```

### 5. C++ thread_local 초기화

C++의 `thread_local`은 동적 초기화 시 오버헤드가 있습니다.

```cpp
// 동적 초기화 - 매 접근마다 초기화 여부 확인
thread_local std::string name = compute_name();

// 더 효율적 - 상수 초기화
thread_local int counter = 0;  // 0으로 초기화는 오버헤드 없음

// 또는 __thread 사용 (C++ extension)
__thread int counter = 0;  // 동적 초기화 불가, 더 빠름
```

---

## 요약

### TLS 핵심 포인트

1. **목적**: 스레드별 독립 데이터로 동기화 없이 스레드 안전성 확보
2. **구현**: FS/GS 세그먼트 레지스터 + TCB + DTV
3. **성능**: 정적 TLS(~1-5ns) >> 동적 TLS(~10-30ns) >> Mutex(~100ns+)

### 사용 가이드

| 상황 | 권장 방식 |
|------|----------|
| 실행 파일 내 빈번한 접근 | `__thread` / `thread_local` |
| 공유 라이브러리 | 동적 TLS (pthread_key) |
| destructor 필요 | 동적 TLS |
| C++11 호환 필요 | `thread_local` |
| 최대 성능 필요 | `__thread` + 로컬 캐싱 |

### 주요 Use Cases

1. **Thread-Local Memory Pool**: TCMalloc, jemalloc 등에서 핵심 기술
2. **errno**: 표준 C 라이브러리의 스레드 안전 에러 처리
3. **Thread-Local Cache**: 시간 캐시, DNS 캐시 등
4. **Request Context**: 웹 서버에서 요청별 컨텍스트 관리
5. **Per-Thread Random**: 락 없는 난수 생성

---

## 참고 자료

- [All about thread-local storage - MaskRay](https://maskray.me/blog/2021-02-14-all-about-thread-local-storage)
- [A Deep dive into (implicit) Thread Local Storage](https://chao-tic.github.io/blog/2018/12/25/tls)
- [ELF Handling For Thread-Local Storage - Ulrich Drepper](https://www.uclibc.org/docs/tls.pdf)
- [Thread Local Storage - OSDev Wiki](https://wiki.osdev.org/Thread_Local_Storage)
- [TCMalloc Design](https://google.github.io/tcmalloc/design.html)
- [Linux Kernel FS/GS Documentation](https://docs.kernel.org/arch/x86/x86_64/fsgs.html)
- [Using FS and GS segments in user space - Linux Kernel](https://docs.kernel.org/arch/x86/x86_64/fsgs.html)

---

## 관련 문서

- [프로세스 vs 스레드](./01-process-vs-thread.md) - 스레드의 메모리 공유 이해
- [컨텍스트 스위칭](./04-context-switching.md) - TLS 전환 과정
- [Thread Pool](../04-concurrency-patterns/03-thread-pool.md) - Thread-Local Pool 응용
- [플랫폼 차이: TLS](../10-platform-differences/03-thread-local-storage.md) - Windows vs POSIX 비교
