# 프로세스 vs 스레드

## 📌 핵심 개념

**프로세스(Process)**는 실행 중인 프로그램의 인스턴스로, 독립적인 메모리 공간과 시스템 자원을 가집니다.

**스레드(Thread)**는 프로세스 내에서 실행되는 **가벼운 실행 단위**로, 같은 프로세스의 다른 스레드와 메모리를 공유합니다.

---

## 🏗️ 메모리 구조 비교

### 프로세스 메모리 구조

```
Process 1                Process 2
┌─────────────┐          ┌─────────────┐
│   Stack     │          │   Stack     │
├─────────────┤          ├─────────────┤
│     ↓       │          │     ↓       │
│  (growth)   │          │  (growth)   │
│             │          │             │
│     ↑       │          │     ↑       │
├─────────────┤          ├─────────────┤
│    Heap     │          │    Heap     │
├─────────────┤          ├─────────────┤
│  Data (BSS) │          │  Data (BSS) │
├─────────────┤          ├─────────────┤
│  Data (Init)│          │  Data (Init)│
├─────────────┤          ├─────────────┤
│    Code     │          │    Code     │
└─────────────┘          └─────────────┘
   독립된 주소 공간           독립된 주소 공간
```

**특징**: 각 프로세스는 완전히 격리된 메모리 공간을 가짐

### 스레드 메모리 구조

```
Single Process with 3 Threads
┌─────────────────────────────────┐
│  Thread 1   Thread 2   Thread 3 │
│  ┌──────┐  ┌──────┐  ┌──────┐  │
│  │Stack │  │Stack │  │Stack │  │ ← 각 스레드 독립
│  │  1   │  │  2   │  │  3   │  │
│  └──────┘  └──────┘  └──────┘  │
├─────────────────────────────────┤
│         Heap (공유)              │ ← 모든 스레드 공유
├─────────────────────────────────┤
│       Data Segment (공유)        │ ← 모든 스레드 공유
├─────────────────────────────────┤
│       Code Segment (공유)        │ ← 모든 스레드 공유
└─────────────────────────────────┘
```

**핵심 차이**:
- **각 스레드 독립**: Stack, Thread-Local Storage (TLS), Registers
- **모든 스레드 공유**: Code, Data, Heap, File Descriptors

---

## 📊 상세 비교표

| 특성 | 프로세스 | 스레드 |
|------|----------|--------|
| **메모리 공간** | 독립적 (완전 격리) | 공유 (같은 프로세스 내) |
| **생성 비용** | 높음 (1-2ms) | 낮음 (0.1ms) |
| **종료 비용** | 높음 | 낮음 |
| **통신 방법** | IPC (Pipe, Socket, Shared Memory) | 공유 메모리 (변수) |
| **통신 속도** | 느림 (커널 개입) | 빠름 (직접 접근) |
| **안정성** | 높음 (크래시 격리) | 낮음 (한 스레드 크래시 시 전체 종료) |
| **자원 소비** | 높음 (각 프로세스마다 메모리) | 낮음 (공유) |
| **Context Switch** | 느림 (TLB flush 등) | 상대적으로 빠름 |
| **동기화** | 불필요 (격리됨) | 필수 (공유 자원) |
| **디버깅** | 쉬움 (격리) | 어려움 (Race Condition) |

---

## 🔍 스레드가 공유하는 것 vs 독립적인 것

### 공유 (Shared)

| 항목 | 설명 |
|------|------|
| **Code Segment** | 프로그램 코드 (함수 등) |
| **Data Segment** | 전역 변수, 정적 변수 |
| **Heap** | 동적 할당 메모리 (`new`, `malloc`) |
| **File Descriptors** | 열린 파일, 소켓 |
| **Signal Handlers** | 시그널 처리기 |
| **Working Directory** | 현재 작업 디렉토리 |
| **User/Group ID** | 사용자 권한 |

### 독립 (Per-Thread)

| 항목 | 설명 |
|------|------|
| **Stack** | 지역 변수, 함수 호출 스택 |
| **Program Counter (PC)** | 현재 실행 중인 명령어 위치 |
| **Registers** | CPU 레지스터 값 |
| **Thread ID** | 스레드 고유 식별자 |
| **Thread-Local Storage (TLS)** | 스레드별 전역 변수 |
| **Errno** | 에러 코드 |
| **Signal Mask** | 블록된 시그널 |

---

## 💻 실전 예시

### 예시 1: 공유 메모리 (스레드)

```cpp
#include <thread>
#include <iostream>

int shared_counter = 0;  // 모든 스레드가 공유

void increment() {
    for (int i = 0; i < 100000; i++) {
        shared_counter++;  // ⚠️ Race Condition 발생!
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter << std::endl;
    // 예상: 200000, 실제: 150000 (Race Condition)
    return 0;
}
```

### 예시 2: 독립 메모리 (프로세스)

```cpp
#include <unistd.h>
#include <iostream>

int shared_counter = 0;  // 각 프로세스마다 독립된 복사본

void increment() {
    for (int i = 0; i < 100000; i++) {
        shared_counter++;
    }
}

int main() {
    pid_t pid = fork();  // 프로세스 생성

    if (pid == 0) {
        // 자식 프로세스
        increment();
        std::cout << "Child: " << shared_counter << std::endl;  // 100000
    } else {
        // 부모 프로세스
        increment();
        wait(nullptr);  // 자식 종료 대기
        std::cout << "Parent: " << shared_counter << std::endl;  // 100000
    }
    return 0;
}
```

**관찰**: 프로세스는 각자 독립된 `shared_counter`를 가지므로 Race Condition 없음

---

## 🔄 프로세스 간 통신 (IPC) vs 스레드 간 통신

### 프로세스 간 통신 (IPC)

| 방법 | 설명 | 속도 | 복잡도 |
|------|------|------|--------|
| **Pipe** | 단방향 데이터 스트림 | 중간 | 낮음 |
| **Named Pipe (FIFO)** | 파일 시스템 기반 양방향 | 중간 | 중간 |
| **Message Queue** | 메시지 단위 전송 | 중간 | 중간 |
| **Shared Memory** | 공유 메모리 영역 | **빠름** | 높음 (동기화 필요) |
| **Socket** | 네트워크 기반 | 느림 | 중간 |
| **Signal** | 이벤트 알림 | 빠름 | 낮음 (데이터 전송 불가) |

### 스레드 간 통신

```cpp
// 단순히 변수를 공유하면 됨 (동기화 필요)
std::mutex mtx;
int shared_data = 0;

void writer() {
    std::lock_guard<std::mutex> lock(mtx);
    shared_data = 42;
}

void reader() {
    std::lock_guard<std::mutex> lock(mtx);
    std::cout << shared_data << std::endl;
}
```

**장점**: 간단하고 빠름 (커널 개입 불필요)
**단점**: 동기화 필요 (Mutex, Atomic 등)

---

## 🏛️ 커널 스레드 vs 유저 스레드

### 커널 스레드 (Kernel-Level Thread)

- **정의**: 운영체제 커널이 직접 관리하는 스레드
- **스케줄링**: 커널 스케줄러가 담당
- **장점**:
  - 멀티코어에서 진정한 병렬 실행
  - 한 스레드가 블로킹되어도 다른 스레드는 실행 가능
- **단점**:
  - 생성/전환 비용 높음 (시스템 콜 필요)

**사용**: Linux pthread, Windows Thread, Java Thread

### 유저 스레드 (User-Level Thread)

- **정의**: 애플리케이션 레벨에서 관리되는 스레드
- **스케줄링**: 유저 공간 라이브러리가 담당
- **장점**:
  - 생성/전환 비용 매우 낮음
  - 커널 개입 불필요
- **단점**:
  - 한 스레드가 블로킹되면 전체 프로세스 블로킹
  - 멀티코어 활용 불가 (커널은 하나의 프로세스로 인식)

**사용**: 초기 Green Thread (현대에는 거의 사용 안함)

### 하이브리드: M:N 모델

- **정의**: M개의 유저 스레드를 N개의 커널 스레드에 매핑
- **장점**: 유저 스레드의 경량성 + 커널 스레드의 병렬성
- **단점**: 스케줄링 복잡도 높음

**사용**: Go의 Goroutine (M:N 스케줄러)

```
Go Goroutines (M:N Model)
┌────────────────────────────────┐
│ G1  G2  G3  G4  G5  G6  G7  G8 │ ← Goroutines (User-level)
└────────────┬───────────────────┘
             │ M:N Mapping
         ┌───┴───┬───┬───┐
         │ M1    │M2 │M3 │         ← Kernel Threads
         └───┬───┴───┴───┘
             │
         ┌───┴─────┬────┬────┐
         │ CPU1    │CPU2│CPU3│     ← Physical CPUs
         └─────────┴────┴────┘
```

---

## 🎯 언제 무엇을 사용할까?

### 프로세스를 사용해야 하는 경우

✅ **안정성이 최우선**
- 브라우저 탭 (한 탭 크래시가 다른 탭에 영향 주면 안됨)
- 서버의 독립된 서비스

✅ **보안/격리 필요**
- 샌드박스 환경
- 서로 다른 권한 필요

✅ **언어 제약**
- Python GIL (Global Interpreter Lock) 회피

✅ **분산 시스템**
- 서로 다른 머신에서 실행

### 스레드를 사용해야 하는 경우

✅ **같은 데이터 공유**
- 게임 서버 (플레이어 데이터 공유)
- 웹 서버 (요청 처리)

✅ **빠른 생성/종료**
- 스레드 풀
- 작업 큐

✅ **메모리 효율**
- 수천~수만 개의 동시 연결 (C10K 문제)

✅ **낮은 레이턴시**
- 실시간 게임 서버
- 금융 거래 시스템

---

## ⚠️ 주의사항

### 프로세스의 함정

1. **높은 메모리 사용**
   - 각 프로세스마다 독립된 메모리 → 중복 데이터

2. **IPC 복잡도**
   - 공유 메모리 설정, 동기화 등 복잡

3. **디버깅 어려움**
   - 여러 프로세스 간 상호작용 추적 어려움

### 스레드의 함정

1. **Race Condition**
   - 공유 데이터 접근 시 동기화 필수

2. **Deadlock**
   - 여러 락 사용 시 교착 상태 발생 가능

3. **전체 크래시**
   - 한 스레드 크래시 시 프로세스 전체 종료

4. **디버깅 악몽**
   - 비결정적 버그 (실행마다 다른 결과)

---

## 📚 예시 코드

### C++: 스레드 생성

```cpp
#include <thread>
#include <iostream>

void print_message(const std::string& msg) {
    std::cout << msg << std::endl;
}

int main() {
    std::thread t1(print_message, "Hello from thread 1");
    std::thread t2(print_message, "Hello from thread 2");

    t1.join();
    t2.join();
    return 0;
}
```

### Linux: 프로세스 생성

```cpp
#include <unistd.h>
#include <iostream>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        std::cout << "Child process, PID: " << getpid() << std::endl;
    } else if (pid > 0) {
        std::cout << "Parent process, PID: " << getpid() << std::endl;
        wait(nullptr);  // 자식 종료 대기
    } else {
        std::cerr << "Fork failed!" << std::endl;
    }
    return 0;
}
```

---

## 🔗 다음 단계

- [동시성 vs 병렬성](./02-concurrency-vs-parallelism.md) - 개념적 차이 이해
- [스레드 생명주기](./03-thread-lifecycle.md) - 스레드 상태 전이
- [동기화 기법](../02-synchronization/README.md) - 스레드 간 안전한 통신

---

## 🔬 내부 메커니즘 심층 분석

### 커널 자료구조

#### Linux: task_struct

Linux에서 프로세스와 스레드는 모두 `task_struct` 구조체로 표현됩니다.

```c
// include/linux/sched.h (단순화)
struct task_struct {
    // 스케줄링 정보
    volatile long state;        // 프로세스 상태
    int prio, static_prio;      // 우선순위

    // 프로세스/스레드 식별
    pid_t pid;                  // Process ID
    pid_t tgid;                 // Thread Group ID (메인 스레드의 PID)

    // 메모리 관리
    struct mm_struct *mm;       // 메모리 디스크립터 (가상 주소 공간)

    // 파일 시스템
    struct fs_struct *fs;       // 현재 디렉토리, 루트 디렉토리
    struct files_struct *files; // 열린 파일 디스크립터 테이블

    // 시그널
    struct signal_struct *signal;
    struct sighand_struct *sighand;

    // 스레드 정보
    struct thread_struct thread; // CPU 레지스터 상태

    // 부모/자식 관계
    struct task_struct *parent;
    struct list_head children;
    struct list_head sibling;

    // 스레드 그룹 (같은 프로세스의 스레드들)
    struct list_head thread_group;
};
```

**핵심 포인트**:
- `pid`: 각 스레드마다 고유 (Linux에서 스레드 = 경량 프로세스)
- `tgid`: Thread Group ID, 같은 프로세스의 모든 스레드가 동일한 값
- `getpid()` 시스템 콜은 실제로 `tgid`를 반환 (POSIX 호환)
- `gettid()` 시스템 콜은 실제 `pid`를 반환

#### Windows: EPROCESS / ETHREAD

```c
// Windows 커널 구조 (단순화)
struct _EPROCESS {
    KPROCESS Pcb;                    // 스케줄링 정보
    HANDLE UniqueProcessId;          // 프로세스 ID
    PVOID VirtualAddress;            // 가상 주소 공간
    HANDLE_TABLE ObjectTable;        // 핸들 테이블 (파일, 동기화 객체 등)
    LIST_ENTRY ThreadListHead;       // 스레드 리스트
    // ...
};

struct _ETHREAD {
    KTHREAD Tcb;                     // 스케줄링 정보
    PVOID StartAddress;              // 스레드 시작 주소
    CLIENT_ID Cid;                   // Process ID + Thread ID
    struct _EPROCESS *Process;       // 소속 프로세스
    // ...
};
```

### clone() 시스템 콜과 플래그

Linux에서 `fork()`와 `pthread_create()`는 모두 내부적으로 `clone()` 시스템 콜을 사용합니다.

```c
// clone() 플래그에 따른 공유 범위 결정
int clone(int (*fn)(void *), void *stack, int flags, void *arg);

// 주요 플래그
#define CLONE_VM      0x00000100  // 메모리 공간 공유
#define CLONE_FS      0x00000200  // 파일 시스템 정보 공유
#define CLONE_FILES   0x00000400  // 파일 디스크립터 테이블 공유
#define CLONE_SIGHAND 0x00000800  // 시그널 핸들러 공유
#define CLONE_THREAD  0x00010000  // 같은 스레드 그룹
```

#### fork() vs pthread_create() 내부 차이

```c
// fork() 호출 시 (프로세스 생성)
clone(NULL, NULL, SIGCHLD, NULL);
// → 모든 것이 복사됨 (Copy-on-Write)

// pthread_create() 호출 시 (스레드 생성)
clone(start_routine, stack,
      CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND |
      CLONE_THREAD | CLONE_SYSVSEM | CLONE_SETTLS |
      CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID,
      arg);
// → 대부분의 자원을 공유
```

### 메모리 레이아웃 상세

```
x86-64 Linux 프로세스 가상 주소 공간 (48-bit)
┌─────────────────────────────────────┐ 0xFFFF_FFFF_FFFF_FFFF
│         Kernel Space (상위)          │
│         (모든 프로세스 공유)          │
├─────────────────────────────────────┤ 0xFFFF_8000_0000_0000
│         Non-canonical hole          │
├─────────────────────────────────────┤ 0x0000_7FFF_FFFF_FFFF
│                                     │
│     Stack (grows down) ↓            │ ← 각 스레드마다 별도
│     [Thread 1 Stack]                │
│     [Thread 2 Stack]                │
│     [Thread 3 Stack]                │
│     ...                             │
│                                     │
├─────────────────────────────────────┤
│     Memory-mapped regions           │ ← mmap, 공유 라이브러리
│     (shared libraries, mmap)        │
├─────────────────────────────────────┤
│                                     │
│     Heap (grows up) ↑               │ ← 모든 스레드 공유
│     (malloc, new)                   │
│                                     │
├─────────────────────────────────────┤
│     BSS (uninitialized data)        │ ← 모든 스레드 공유
├─────────────────────────────────────┤
│     Data (initialized data)         │ ← 모든 스레드 공유
├─────────────────────────────────────┤
│     Text (code)                     │ ← 모든 스레드 공유, Read-only
├─────────────────────────────────────┤ 0x0000_0000_0040_0000
│         Reserved                    │
└─────────────────────────────────────┘ 0x0000_0000_0000_0000
```

### 스레드 스택 할당

```c
// pthread 스택 할당 내부 동작
void *pthread_stack_allocation(size_t size) {
    // 1. 스택 크기 결정 (기본 8MB on Linux)
    size_t stack_size = size ? size : PTHREAD_STACK_DEFAULT;

    // 2. Guard page 포함하여 메모리 매핑
    void *stack = mmap(NULL,
                       stack_size + GUARD_SIZE,
                       PROT_READ | PROT_WRITE,
                       MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK,
                       -1, 0);

    // 3. Guard page 설정 (스택 오버플로우 감지)
    mprotect(stack, GUARD_SIZE, PROT_NONE);

    return stack + GUARD_SIZE;  // 실제 스택 시작점
}
```

```
Thread Stack Layout
┌───────────────────┐ 높은 주소
│   Thread Local    │
│    Storage (TLS)  │
├───────────────────┤
│                   │
│   Stack (사용 중)  │
│        ↓          │
│   (grows down)    │
│                   │
│   Stack (미사용)   │
│                   │
├───────────────────┤
│   Guard Page      │ ← PROT_NONE, 접근 시 SIGSEGV
│   (보호 페이지)    │
└───────────────────┘ 낮은 주소
```

### 컨텍스트 스위칭 비용 분석

```c
// 스레드 컨텍스트 (task_struct.thread)
struct thread_struct {
    // x86-64 레지스터 상태
    unsigned long sp;     // Stack Pointer
    unsigned long ip;     // Instruction Pointer
    unsigned long fs;     // TLS base (FS segment)
    unsigned long gs;     // Per-CPU data (kernel)

    // FPU/SSE/AVX 상태 (Lazy saving)
    struct fpu fpu;

    // 디버그 레지스터
    unsigned long debugreg[8];
};
```

**스레드 vs 프로세스 컨텍스트 스위칭 비용**:

| 항목 | 스레드 전환 | 프로세스 전환 |
|------|-------------|---------------|
| 레지스터 저장/복원 | ✓ (~100 cycles) | ✓ (~100 cycles) |
| 스택 포인터 전환 | ✓ (~10 cycles) | ✓ (~10 cycles) |
| TLB Flush | ✗ 불필요 | ✓ 필요 (~1000+ cycles) |
| 캐시 무효화 | 부분적 | 전체 가능 |
| 페이지 테이블 전환 | ✗ 불필요 | ✓ 필요 (CR3 레지스터) |
| **총 비용** | ~1-2 μs | ~3-5 μs |

**PCID (Process Context ID) 최적화**:
```
Intel Haswell 이후 지원 (CR4.PCIDE = 1)
- TLB 엔트리에 12비트 PCID 태그 추가
- 프로세스 전환 시 TLB flush 불필요
- 다른 프로세스 엔트리는 PCID로 구분
→ 프로세스 전환 비용 크게 감소
```

### /proc 파일시스템으로 확인

```bash
# 프로세스 정보
$ cat /proc/[pid]/status
Name:   myprogram
Pid:    12345
Tgid:   12345
Threads: 4

# 스레드 목록 (같은 프로세스 내)
$ ls /proc/12345/task/
12345  12346  12347  12348

# 각 스레드의 스택 주소
$ cat /proc/12345/task/12346/maps | grep stack
7f1234560000-7f1234580000 rw-p 00000000 00:00 0 [stack:12346]

# 메모리 맵 (공유 확인)
$ cat /proc/12345/maps
00400000-00452000 r-xp ... /myprogram        # Code (공유)
00651000-00652000 rw-p ... /myprogram        # Data (공유)
7f1234500000-7f1234520000 rw-p ... [heap]    # Heap (공유)
7f1234560000-7f1234580000 rw-p ... [stack:12346] # Thread stack (독립)
```

---

## 📖 참고 자료

- "Operating Systems: Three Easy Pieces" - Chapter 26 (Threads)
- "The Linux Programming Interface" - Chapter 28 (Processes)
- Linux Kernel Source: `include/linux/sched.h`
- "Understanding the Linux Kernel" - Bovet & Cesati
- POSIX Threads Programming: https://computing.llnl.gov/tutorials/pthreads/

---

*프로세스와 스레드의 차이를 이해하는 것은 멀티스레드 프로그래밍의 첫 걸음입니다.*
