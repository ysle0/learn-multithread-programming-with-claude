# 컨텍스트 스위칭

## 📌 핵심 개념

**컨텍스트 스위칭(Context Switching)**은 CPU가 한 스레드/프로세스에서 다른 스레드/프로세스로 전환할 때 발생하는 작업입니다. 이는 멀티태스킹의 핵심이지만, **성능 오버헤드**가 큽니다.

---

## 🔄 컨텍스트 스위칭 과정

### 1. 전체 과정

```
Thread A 실행 중
    ↓
1. 인터럽트 발생 (Timer, I/O)
    ↓
2. Thread A의 컨텍스트 저장
   - CPU Registers
   - Program Counter (PC)
   - Stack Pointer (SP)
    ↓
3. Thread B의 컨텍스트 복원
   - Registers 로드
   - PC, SP 복원
    ↓
4. Thread B 실행 재개
```

### 2. 상세 단계

```
┌─────────────────────────────────────────┐
│ 1. Interrupt Handler 진입               │
│    - Timer interrupt                    │
│    - I/O complete                       │
│    - System call                        │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 2. 현재 스레드(A) 상태 저장             │
│    a) Registers → PCB (Process Control  │
│       Block)                            │
│    b) Program Counter                   │
│    c) Stack Pointer                     │
│    d) CPU flags                         │
│    e) FPU state (부동소수점)            │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 3. 스케줄러 실행                        │
│    - 다음 실행할 스레드(B) 선택         │
│    - 스케줄링 알고리즘 적용             │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 4. 다음 스레드(B) 상태 복원             │
│    a) PCB → Registers                   │
│    b) Program Counter 복원              │
│    c) Stack Pointer 복원                │
│    d) MMU 설정 (페이지 테이블)          │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 5. 캐시/TLB 무효화                      │
│    - L1/L2 캐시 미스 발생               │
│    - TLB (Translation Lookaside Buffer) │
│      flush                              │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│ 6. 스레드 B 실행 재개                   │
│    - 이전 중단 지점부터 계속            │
└─────────────────────────────────────────┘
```

---

## 💾 저장/복원되는 컨텍스트

### CPU Registers

| Register | 설명 | 크기 |
|----------|------|------|
| **General Purpose** | RAX, RBX, RCX, RDX, RSI, RDI, R8-R15 | 64-bit × 16 |
| **Program Counter** | RIP (다음 실행 명령어 주소) | 64-bit |
| **Stack Pointer** | RSP (스택 최상단) | 64-bit |
| **Base Pointer** | RBP (스택 프레임) | 64-bit |
| **Flags** | RFLAGS (상태 플래그) | 64-bit |
| **Segment Registers** | CS, DS, SS, ES, FS, GS | 16-bit × 6 |
| **FPU Registers** | x87 FPU, SSE, AVX | 수백 바이트 |

**총 크기**: 약 1-2KB

### Process Control Block (PCB)

```c
struct PCB {
    // Process ID
    pid_t pid;

    // CPU State
    uint64_t registers[16];   // General purpose
    uint64_t rip;             // Program counter
    uint64_t rsp;             // Stack pointer
    uint64_t rflags;          // CPU flags
    uint8_t fpu_state[512];   // FPU/SSE state

    // Memory Management
    page_table_t* page_table; // 가상 메모리 매핑

    // Scheduling
    int priority;
    int state;                // RUNNING, BLOCKED, etc.
    uint64_t cpu_time;        // 누적 CPU 시간

    // I/O
    file_descriptor_table_t* files;
    // ...
};
```

---

## ⏱️ 컨텍스트 스위칭 비용

### 직접 비용 (Direct Cost)

| 작업 | 시간 |
|------|------|
| **레지스터 저장** | ~100 cycles (~50 ns @ 2GHz) |
| **레지스터 복원** | ~100 cycles |
| **스케줄러 실행** | ~500 cycles |
| **MMU 설정** | ~200 cycles |
| **총 직접 비용** | **~1,000 cycles (~0.5 μs)** |

### 간접 비용 (Indirect Cost)

| 항목 | 영향 |
|------|------|
| **캐시 미스** | 10-100 μs |
| **TLB 미스** | 10-100 ns per miss |
| **Pipeline Flush** | 10-20 cycles |
| **Branch Prediction 초기화** | 수십 cycles |
| **총 간접 비용** | **수십~수백 μs** |

### 실제 측정값

| 시스템 | 컨텍스트 스위칭 시간 |
|--------|---------------------|
| **Linux (x86-64)** | 3-5 μs |
| **Windows** | 5-10 μs |
| **macOS** | 4-8 μs |
| **실시간 OS** | 1-2 μs |

**참고**: 프로세스 간 전환이 스레드 간 전환보다 느림 (TLB flush 등)

---

## 🧊 캐시와 TLB의 영향

### 캐시 계층 구조

```
CPU Core
    ↓
┌─────────────┐
│ L1 Cache    │ ← 32-64 KB, ~4 cycles
│ (32 KB)     │
└─────────────┘
    ↓
┌─────────────┐
│ L2 Cache    │ ← 256 KB-1 MB, ~12 cycles
│ (256 KB)    │
└─────────────┘
    ↓
┌─────────────┐
│ L3 Cache    │ ← 4-32 MB, ~40 cycles (공유)
│ (8 MB)      │
└─────────────┘
    ↓
┌─────────────┐
│ Main Memory │ ← ~200 cycles
│ (16 GB)     │
└─────────────┘
```

### 캐시 미스 시나리오

```
Thread A 실행:
- A의 데이터가 L1/L2 캐시에 로드됨
- 캐시 히트율 90%+

컨텍스트 스위칭 → Thread B

Thread B 실행:
- B의 데이터가 캐시에 없음 (Cold Cache)
- 처음 수천 개 접근은 캐시 미스
- L1 미스 → L2 조회 (~12 cycles)
- L2 미스 → L3 조회 (~40 cycles)
- L3 미스 → 메모리 조회 (~200 cycles)

성능 저하: 수십 μs
```

### TLB (Translation Lookaside Buffer)

**정의**: 가상 주소 → 물리 주소 변환 캐시

```
가상 주소 변환:
1. TLB 조회 (1-2 cycles)
2. TLB 히트 → 물리 주소 반환
3. TLB 미스 → Page Table Walk (수십~수백 cycles)
```

**컨텍스트 스위칭 시**:
- 프로세스 전환: TLB 완전 flush (모든 엔트리 무효화)
- 스레드 전환: TLB 유지 (같은 주소 공간)

**TLB 미스 비용**:
- 단일 미스: 100-200 cycles
- 스위칭 후 수천 개 미스 누적 → 수십 μs

---

## 📊 측정 방법

### Linux: `vmstat`

```bash
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 123456  78910 234567    0    0     0     5 1234 5678 10  5 85  0  0
                                                          ↑    ↑
                                           interrupts    context switches
```

- `cs`: 초당 컨텍스트 스위칭 횟수

### C++ 코드로 측정

```cpp
#include <chrono>
#include <sched.h>
#include <iostream>

void measure_context_switch() {
    const int iterations = 1000000;

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < iterations; i++) {
        sched_yield();  // 명시적으로 CPU 양보 → 컨텍스트 스위칭 유발
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start);

    double avg_ns = duration.count() / (double)iterations;
    std::cout << "Average context switch: " << avg_ns << " ns" << std::endl;
}
```

**예상 출력**: 3,000-10,000 ns (3-10 μs)

---

## 🎯 컨텍스트 스위칭 최소화 전략

### 1. 적절한 스레드 수 유지

**원칙**: 스레드 수 ≈ CPU 코어 수

```cpp
// ❌ 나쁜 예: 과도한 스레드
for (int i = 0; i < 10000; i++) {
    std::thread t(task);
    t.detach();
}
// Context switching 폭증!

// ✅ 좋은 예: 스레드 풀
ThreadPool pool(std::thread::hardware_concurrency());
for (int i = 0; i < 10000; i++) {
    pool.enqueue(task);
}
```

**최적 스레드 수**:
- CPU-bound: 코어 수
- I/O-bound: 코어 수 × (1 + 대기시간/CPU시간)

### 2. CPU Affinity 설정

**개념**: 스레드를 특정 CPU 코어에 고정

**장점**:
- 캐시 친화성 향상 (같은 코어에서 계속 실행)
- TLB 미스 감소

```cpp
#include <pthread.h>

void set_cpu_affinity(int cpu_id) {
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);

    pthread_t thread = pthread_self();
    pthread_setaffinity_np(thread, sizeof(cpu_set_t), &cpuset);
}

void worker() {
    set_cpu_affinity(2);  // CPU 2번에 고정
    // 작업 수행
}
```

### 3. Lock 경합 최소화

**문제**: Lock 경합 → 스레드 블로킹 → 컨텍스트 스위칭

```cpp
// ❌ 나쁜 예: 긴 Critical Section
std::mutex mtx;
void process() {
    std::lock_guard<std::mutex> lock(mtx);
    expensive_computation();  // 긴 작업
    more_work();
}

// ✅ 좋은 예: Critical Section 최소화
std::mutex mtx;
void process() {
    auto result = expensive_computation();  // 락 밖에서 계산

    std::lock_guard<std::mutex> lock(mtx);
    update_shared_data(result);  // 최소한만 보호
}
```

### 4. Spinlock 사용 (짧은 Critical Section)

**개념**: 락을 기다리며 Busy-Waiting (블로킹 안함)

**적용 조건**:
- Critical Section이 매우 짧음 (<100 ns)
- 컨텍스트 스위칭 비용 > Busy-Wait 비용

```cpp
#include <atomic>

class Spinlock {
    std::atomic_flag flag = ATOMIC_FLAG_INIT;

public:
    void lock() {
        while (flag.test_and_set(std::memory_order_acquire)) {
            // Busy-wait (컨텍스트 스위칭 없음)
            #ifdef __x86_64__
            __builtin_ia32_pause();  // CPU 힌트
            #endif
        }
    }

    void unlock() {
        flag.clear(std::memory_order_release);
    }
};
```

**주의**: Critical Section이 길면 CPU 낭비!

### 5. 비동기 I/O 사용

**문제**: 동기 I/O → 블로킹 → 컨텍스트 스위칭

```cpp
// ❌ 동기 I/O: 블로킹
std::ifstream file("data.txt");
std::string line;
std::getline(file, line);  // 블로킹 → 컨텍스트 스위칭

// ✅ 비동기 I/O (예: io_uring, libuv)
async_read_file("data.txt", [](std::string data) {
    // 콜백: I/O 완료 시 호출
    process(data);
});
// 다른 작업 계속 가능
```

---

## 📈 성능 영향 분석

### 예시: 웹 서버

**시나리오**: 10,000 동시 연결, 요청당 1ms 처리

| 방식 | 컨텍스트 스위칭 | 처리량 |
|------|----------------|--------|
| **Thread-per-Request** | 초당 수만~수십만 회 | 낮음 (오버헤드 큼) |
| **Thread Pool (10 threads)** | 초당 수천 회 | 높음 |
| **Async I/O (단일 스레드)** | 초당 수십 회 | **매우 높음** |

**관찰**: 비동기 I/O가 컨텍스트 스위칭을 대폭 줄여 성능 향상

### 벤치마크: 스레드 수 vs 성능

```
CPU: 4 코어
작업: CPU-bound (계산 집약적)

스레드 수    처리량    컨텍스트 스위칭/초
1            100%      ~10
2            180%      ~50
4            350%      ~200
8            320%      ~5,000   ← 오버헤드 증가
16           280%      ~20,000  ← 성능 저하
32           200%      ~50,000  ← 심각한 저하
```

**결론**: 코어 수를 넘어서면 오히려 성능 저하

---

## ⚠️ 일반적인 실수

### 1. 과도한 스레드 생성

```cpp
// ❌ 매 요청마다 스레드 생성
void handle_request(Request req) {
    std::thread t([req]() {
        process(req);
    });
    t.detach();
}
```

**문제**: 수천 개 스레드 → Context Switching 폭증

**해결**: Thread Pool

### 2. 불필요한 Yield

```cpp
// ❌ 의미 없는 yield
while (!ready) {
    std::this_thread::yield();  // Busy-wait + Context Switching
}

// ✅ Condition Variable 사용
std::unique_lock<std::mutex> lock(mtx);
cv.wait(lock, [] { return ready; });
```

### 3. 잦은 Lock/Unlock

```cpp
// ❌ 루프에서 반복적인 락
for (int i = 0; i < 1000000; i++) {
    mtx.lock();
    shared_data++;
    mtx.unlock();  // 매번 컨텍스트 스위칭 가능
}

// ✅ 배치 처리
mtx.lock();
for (int i = 0; i < 1000000; i++) {
    shared_data++;
}
mtx.unlock();
```

---

## 🔗 다음 단계

- [동기화 기법](../02-synchronization/README.md) - Lock, Semaphore 등
- [동시성 패턴](../04-concurrency-patterns/README.md) - Thread Pool 등
- [Lock-Free 프로그래밍](../05-lock-free-programming/README.md) - 블로킹 없는 동기화

---

## 🔬 내부 메커니즘 심층 분석

### x86-64 컨텍스트 저장 구조

```c
// Linux 커널: arch/x86/include/asm/processor.h
struct thread_struct {
    // 스케줄링 관련
    unsigned long sp;           // Stack Pointer (RSP)
    unsigned short es, ds;      // 세그먼트 레지스터

    // TLS (Thread Local Storage)
    unsigned long fsbase;       // FS 베이스 주소 (user TLS)
    unsigned long gsbase;       // GS 베이스 주소 (kernel per-CPU)

    // 디버그 레지스터
    unsigned long debugreg0;
    unsigned long debugreg1;
    unsigned long debugreg2;
    unsigned long debugreg3;
    unsigned long debugreg6;
    unsigned long debugreg7;

    // FPU 상태 (lazy saving)
    struct fpu fpu;
    // ...
};
```

### FPU/SIMD 상태의 Lazy Saving

```
FPU 상태 저장 최적화 (Lazy FPU):

기존 방식:
┌─────────────────────────────────────────────────────┐
│ Context Switch 시 항상 FPU 상태 저장/복원           │
│ - FXSAVE: 512 bytes (SSE)                          │
│ - XSAVE: 2KB+ (AVX-512)                            │
│ → 비용: 수백 cycles                                 │
└─────────────────────────────────────────────────────┘

Lazy 방식 (현대 Linux):
┌─────────────────────────────────────────────────────┐
│ 1. Context Switch 시 FPU 상태 저장 건너뜀          │
│ 2. CR0.TS (Task Switched) 플래그 설정              │
│ 3. 새 스레드가 FPU 사용 시 #NM 예외 발생           │
│ 4. 예외 핸들러에서:                                │
│    - 이전 스레드 FPU 저장                          │
│    - 현재 스레드 FPU 복원                          │
│    - CR0.TS 클리어                                 │
│                                                    │
│ 장점: FPU 미사용 스레드 → 저장/복원 비용 0         │
└─────────────────────────────────────────────────────┘

최신 방식 (XSAVEOPT, 2018+):
- Eager FPU: 항상 저장하지만 XSAVEOPT/XSAVEC로 최적화
- 변경된 컴포넌트만 저장 (Intel MPX, AVX state 등)
```

### 실제 Context Switch 코드 (Linux 커널)

```c
// arch/x86/kernel/process_64.c (단순화)
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
    struct thread_struct *prev = &prev_p->thread;
    struct thread_struct *next = &next_p->thread;

    // 1. FPU 상태 저장/복원
    switch_fpu_prepare(prev_p, cpu);
    switch_fpu_finish(next_p, cpu);

    // 2. TLS 전환 (FS/GS 베이스)
    savesegment(gs, prev->gsindex);
    load_gs_index(next->gsindex);
    wrmsrl(MSR_FS_BASE, next->fsbase);  // User TLS
    wrmsrl(MSR_KERNEL_GS_BASE, next->gsbase);

    // 3. 스택 전환
    // 이 시점에서 RSP가 새 스레드의 커널 스택으로 전환

    // 4. 디버그 레지스터 (필요시)
    if (unlikely(next->debugreg7))
        load_debugregs(next);

    return prev_p;
}
```

### PCID (Process-Context Identifier) 상세

```
PCID 없이 (구형 CPU):
┌────────────────────────────────────────────────────┐
│ 프로세스 A → 프로세스 B 전환                        │
│                                                    │
│ 1. CR3 레지스터에 새 페이지 테이블 로드            │
│ 2. TLB 전체 flush (INVLPG 또는 CR3 reload)        │
│                                                    │
│ TLB 미스 폭증:                                     │
│ - 프로세스 B의 첫 ~1000개 메모리 접근이 TLB miss  │
│ - 각 miss: 4-level page walk = ~100 cycles        │
│ - 총 비용: ~100,000 cycles = ~50μs               │
└────────────────────────────────────────────────────┘

PCID 사용 (Intel Haswell 이후):
┌────────────────────────────────────────────────────┐
│ CR4.PCIDE = 1 활성화                               │
│                                                    │
│ TLB 엔트리 구조:                                   │
│ ┌─────────┬────────┬──────────┬─────────┐        │
│ │ Virtual │ Physical│ Flags   │ PCID    │        │
│ │ Address │ Address │         │ (12-bit)│        │
│ └─────────┴────────┴──────────┴─────────┘        │
│                                                    │
│ 프로세스 A (PCID=1) → 프로세스 B (PCID=2) 전환    │
│ 1. CR3 = B의 페이지 테이블 | PCID=2 | NOFLUSH    │
│ 2. TLB flush 없음! (PCID로 구분)                  │
│ 3. 다시 A로 돌아오면 TLB 히트                     │
│                                                    │
│ 성능 향상: ~50% 컨텍스트 스위칭 비용 감소         │
└────────────────────────────────────────────────────┘
```

```bash
# PCID 지원 확인
$ grep pcid /proc/cpuinfo
flags : ... pcid ...

# PCID 사용 확인 (커널 부팅 로그)
$ dmesg | grep PCID
[    0.000000] x86/mm: PCID enabled
```

### 직접 vs 간접 비용 상세 분석

```
Context Switch 비용 분해 (x86-64, ~3GHz CPU):

[직접 비용 - 피할 수 없음]
┌──────────────────────────────────────────────────┐
│ 레지스터 저장 (범용 16개)     │   ~50 cycles    │
│ 레지스터 복원                 │   ~50 cycles    │
│ 스케줄러 결정                 │  ~200 cycles    │
│ 스택 전환                     │   ~30 cycles    │
│ TLS 전환 (FS/GS base)        │   ~50 cycles    │
│                               │                 │
│ 소계                          │  ~400 cycles    │
│                               │   ~130 ns       │
└──────────────────────────────────────────────────┘

[간접 비용 - 가변적]
┌──────────────────────────────────────────────────┐
│ TLB miss (프로세스 전환)                          │
│ - 페이지 수에 비례                               │
│ - PCID 있으면 크게 감소                          │
│ - 최악: ~100,000 cycles                          │
│                                                  │
│ L1 캐시 miss                                     │
│ - Cold cache 상태                                │
│ - 워킹셋 크기에 비례                             │
│ - 최악: ~50,000 cycles                           │
│                                                  │
│ L2/L3 캐시 miss                                  │
│ - 더 큰 워킹셋                                   │
│ - L3는 공유되어 덜 심각                          │
│                                                  │
│ Branch predictor 무효화                           │
│ - 새 코드 경로 학습                              │
│ - ~1,000 cycles                                  │
│                                                  │
│ 소계                          │ ~10,000-200,000 │
│                               │ cycles          │
│                               │ 3-70 μs         │
└──────────────────────────────────────────────────┘
```

### perf로 정밀 측정

```bash
# Context Switch 횟수 측정
$ perf stat -e context-switches,cpu-migrations ./program
 Performance counter stats for './program':
             5,234      context-switches
                42      cpu-migrations

# Context Switch 시간 분포 측정
$ perf sched record ./program
$ perf sched latency
 Task                  | Runtime ms | Switches | Average delay
 worker-thread         |    1234.56 |     500  |   0.023 ms

# 스케줄러 이벤트 상세
$ perf sched timehist
           time    cpu  task name       wait time  sch delay   run time
      1.000000 [001]  worker-1          0.000 ms    0.015 ms    1.234 ms
      2.234000 [001]  worker-2          1.234 ms    0.023 ms    0.567 ms
```

### Voluntary vs Involuntary 스위칭

```bash
# /proc/[pid]/status에서 확인
$ cat /proc/self/status | grep ctxt
voluntary_ctxt_switches:        15
nonvoluntary_ctxt_switches:     3

# 의미:
# - voluntary: 스레드가 자발적 양보 (I/O 대기, sleep, mutex 대기)
# - nonvoluntary: 스케줄러 강제 선점 (time slice 만료)

# 높은 nonvoluntary = CPU bound 작업 (정상)
# 높은 voluntary = I/O bound 또는 과도한 동기화
```

### KPTI (Kernel Page Table Isolation) 영향

```
Meltdown 대응 KPTI 활성화 시:
┌────────────────────────────────────────────────────┐
│ User space와 Kernel space 페이지 테이블 분리       │
│                                                    │
│ 시스템 콜/인터럽트 시:                             │
│ 1. CR3 전환 (User PT → Kernel PT)                 │
│ 2. TLB 일부 flush                                 │
│ 3. 처리 완료                                       │
│ 4. CR3 전환 (Kernel PT → User PT)                 │
│ 5. TLB 일부 flush                                 │
│                                                    │
│ 추가 비용: ~100-200 cycles per syscall            │
│ Context switch 영향: ~5-10% 추가 오버헤드         │
│                                                    │
│ PCID + INVPCID로 완화:                            │
│ - flush 범위 최소화                               │
│ - 실제 오버헤드 ~2-5%로 감소                      │
└────────────────────────────────────────────────────┘
```

```bash
# KPTI 상태 확인
$ cat /sys/devices/system/cpu/vulnerabilities/meltdown
Mitigation: PTI

# 부팅 시 비활성화 (보안 위험!)
# 커널 파라미터: nopti
```

---

## 📚 참고 자료

- "Operating Systems: Three Easy Pieces" - Chapter 6 (Mechanism: Limited Direct Execution)
- [Linux Performance Tools](http://www.brendangregg.com/linuxperf.html)
- Linux Kernel Source: `arch/x86/kernel/process_64.c`
- Intel® 64 and IA-32 Architectures Software Developer's Manual - Volume 3

---

*컨텍스트 스위칭은 필요하지만 비용이 큽니다. 최소화하는 것이 성능 최적화의 핵심입니다!*
