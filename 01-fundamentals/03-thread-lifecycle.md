# 스레드 생명주기

## 📌 핵심 개념

스레드는 생성부터 종료까지 여러 **상태(State)**를 거치며, 운영체제 스케줄러가 이러한 상태 전이를 관리합니다. 스레드 생명주기를 이해하면 블로킹, 대기, 성능 문제를 진단할 수 있습니다.

---

## 🔄 스레드 상태 다이어그램

```
         create()
            ↓
       ┌────────┐
       │  NEW   │ ← 생성됨, 아직 시작 안됨
       └────────┘
            ↓ start()
       ┌─────────────┐
       │  RUNNABLE   │ ← 실행 대기 중
       └─────────────┘
            ↓ ↑
      스케줄러 선택
            ↓ ↑
       ┌─────────────┐
       │   RUNNING   │ ← 실제로 CPU에서 실행 중
       └─────────────┘
            │ ↑
            │ ↑ I/O 완료, Lock 획득
            ↓ │
       ┌─────────────┐
       │   BLOCKED   │ ← I/O 대기, Lock 대기, sleep
       └─────────────┘
            │
            │ terminate()
            ↓
       ┌─────────────┐
       │ TERMINATED  │ ← 종료됨
       └─────────────┘
```

---

## 📋 각 상태 상세 설명

### 1. NEW (생성)

**정의**: 스레드 객체가 생성되었지만 아직 `start()`가 호출되지 않은 상태

**특징**:
- 메모리 할당 완료
- 아직 스케줄러에 등록되지 않음
- OS 리소스 미할당

**예시**:
```cpp
std::thread t(worker_function);  // NEW 상태
// 아직 실행 안됨
```

```java
Thread t = new Thread(() -> {
    System.out.println("Work");
});
// NEW 상태 (start() 호출 전)
```

### 2. RUNNABLE (실행 가능)

**정의**: `start()`가 호출되어 **실행 대기 큐**에 들어간 상태

**특징**:
- 스케줄러에 등록됨
- CPU를 할당받기를 대기
- 언제든지 RUNNING으로 전환 가능

**중요**: Java에서는 RUNNABLE과 RUNNING을 구분하지 않지만, OS 레벨에서는 구분됨

**예시**:
```cpp
std::thread t(worker_function);
t.start();  // RUNNABLE 상태로 전환
// 스케줄러가 CPU를 할당할 때까지 대기
```

### 3. RUNNING (실행 중)

**정의**: 스케줄러가 스레드를 선택하여 **실제로 CPU에서 실행** 중인 상태

**특징**:
- CPU 코어를 점유
- 명령어 실제 실행
- Time slice가 끝나거나 블로킹되면 다른 상태로 전환

**전환 조건**:
- **Time slice 만료** → RUNNABLE (다른 스레드에게 양보)
- **I/O 요청** → BLOCKED
- **Lock 획득 대기** → BLOCKED
- **sleep() 호출** → BLOCKED
- **완료** → TERMINATED

### 4. BLOCKED (대기/블로킹)

**정의**: 어떤 이벤트를 기다리며 **실행 불가능**한 상태

**블로킹 원인**:

| 원인 | 설명 | 예시 |
|------|------|------|
| **I/O 대기** | 디스크/네트워크 읽기/쓰기 | `read()`, `write()` |
| **Lock 대기** | Mutex, Semaphore 획득 대기 | `mutex.lock()` |
| **Sleep** | 명시적 대기 | `sleep(1000)` |
| **조건 변수** | 신호 대기 | `cv.wait()` |
| **Join** | 다른 스레드 종료 대기 | `t.join()` |

**세부 상태** (운영체제마다 다름):
- **WAITING**: 무한정 대기 (`wait()`, `join()`)
- **TIMED_WAITING**: 시간 제한 대기 (`sleep()`, `wait(timeout)`)
- **BLOCKED**: Lock 획득 대기

**예시**:
```cpp
std::mutex mtx;

void worker() {
    mtx.lock();  // 다른 스레드가 락 보유 중이면 BLOCKED
    // Critical Section
    mtx.unlock();
}
```

### 5. TERMINATED (종료)

**정의**: 스레드 실행이 **완료**된 상태

**종료 원인**:
- 정상 완료 (함수 return)
- 예외 발생
- 강제 종료 (비권장)

**특징**:
- 더 이상 스케줄링되지 않음
- OS 리소스 해제 대기
- `join()`으로 완료 확인 가능

**예시**:
```cpp
void worker() {
    std::cout << "Work done" << std::endl;
    return;  // TERMINATED 상태로 전환
}

std::thread t(worker);
t.join();  // TERMINATED될 때까지 대기
```

---

## ⏱️ 상태 전이 시나리오

### 시나리오 1: 정상 실행 흐름

```
1. NEW          : std::thread t(worker);
2. RUNNABLE     : (자동으로 스케줄러에 등록)
3. RUNNING      : 스케줄러가 CPU 할당
4. RUNNABLE     : Time slice 만료, 양보
5. RUNNING      : 다시 스케줄링됨
6. TERMINATED   : 작업 완료, return
```

### 시나리오 2: I/O 블로킹

```
1. RUNNING      : 실행 중
2. BLOCKED      : read() 호출, 디스크 I/O 대기
3. RUNNABLE     : I/O 완료, 실행 대기 큐로 복귀
4. RUNNING      : 스케줄러가 다시 선택
5. TERMINATED   : 작업 완료
```

### 시나리오 3: Lock 경합

```
Thread A:
1. RUNNING      : mutex.lock() 성공
2. RUNNING      : Critical Section 실행
3. RUNNING      : mutex.unlock()
4. TERMINATED

Thread B:
1. RUNNING      : mutex.lock() 시도
2. BLOCKED      : Thread A가 락 보유 중, 대기
3. RUNNABLE     : Thread A가 락 해제, 대기 큐로
4. RUNNING      : 락 획득, Critical Section 진입
5. TERMINATED
```

---

## 🔍 스케줄링 알고리즘

운영체제 스케줄러가 RUNNABLE 스레드 중 어느 것을 RUNNING으로 전환할지 결정하는 알고리즘입니다.

### 1. FCFS (First-Come, First-Served)

**원리**: 먼저 도착한 스레드를 먼저 실행

**장점**: 구현 간단, 공정함
**단점**: Convoy Effect (짧은 작업이 긴 작업을 기다림)

```
Queue: [T1(10s)] [T2(1s)] [T3(1s)]
실행: T1(10s) → T2(1s) → T3(1s)
평균 대기: (0 + 10 + 11) / 3 = 7초
```

### 2. Round-Robin (RR)

**원리**: 각 스레드에 동일한 Time Slice 부여, 순환

**장점**: 공정, 응답성 좋음
**단점**: Context Switching 오버헤드

```
Time Slice: 2초
Queue: [T1(5s)] [T2(3s)] [T3(2s)]

실행:
T1(2s) → T2(2s) → T3(2s, 완료) → T1(2s) → T2(1s, 완료) → T1(1s, 완료)
```

### 3. Priority Scheduling

**원리**: 우선순위 높은 스레드를 먼저 실행

**장점**: 중요한 작업 우선 처리
**단점**: Starvation (낮은 우선순위 스레드가 계속 대기)

```
Priority: T1(10), T2(5), T3(1)
실행 순서: T1 → T2 → T3 (우선순위 순)
```

**Starvation 방지**: Priority Aging (대기 시간에 따라 우선순위 증가)

### 4. Multilevel Feedback Queue (MLFQ)

**원리**: 여러 우선순위 큐, 작업 특성에 따라 이동

**특징**:
- I/O-bound: 높은 우선순위
- CPU-bound: 낮은 우선순위로 강등
- 현대 OS (Linux CFS, Windows)의 기반

---

## 🎓 선점형 vs 비선점형 스케줄링

### 선점형 (Preemptive)

**정의**: 스케줄러가 **강제로** 실행 중인 스레드를 중단시키고 다른 스레드 실행 가능

**특징**:
- 현대 OS의 기본 방식
- Time slice 만료 시 자동 전환
- 응답성 우수

**예시**: Linux, Windows, macOS

```
Thread A 실행 중
→ Time slice 만료
→ 스케줄러가 강제로 중단
→ Thread B 실행
```

### 비선점형 (Non-Preemptive)

**정의**: 스레드가 **자발적으로** CPU를 양보할 때까지 실행

**특징**:
- 스레드가 완료되거나 명시적으로 양보할 때만 전환
- 응답성 낮음
- 오래된 시스템에서 사용

**예시**: Windows 3.1 (Cooperative Multitasking)

```
Thread A 실행 중
→ Thread A가 yield() 호출
→ Thread B 실행
```

---

## 💻 코드로 보는 상태 전이

### C++ 예시

```cpp
#include <thread>
#include <mutex>
#include <chrono>
#include <iostream>

std::mutex mtx;

void worker(int id) {
    std::cout << "[" << id << "] RUNNABLE" << std::endl;

    // RUNNING → BLOCKED (Lock 대기)
    mtx.lock();
    std::cout << "[" << id << "] RUNNING (Lock acquired)" << std::endl;

    // RUNNING → BLOCKED (Sleep)
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "[" << id << "] RUNNING (After sleep)" << std::endl;

    mtx.unlock();

    // TERMINATED
    std::cout << "[" << id << "] TERMINATED" << std::endl;
}

int main() {
    std::thread t1(worker, 1);  // NEW → RUNNABLE
    std::thread t2(worker, 2);

    t1.join();  // BLOCKED (t1 종료 대기)
    t2.join();

    return 0;
}
```

### Java 예시

```java
class Worker implements Runnable {
    private final Object lock;
    private final int id;

    public Worker(Object lock, int id) {
        this.lock = lock;
        this.id = id;
    }

    @Override
    public void run() {
        System.out.println("[" + id + "] State: " +
            Thread.currentThread().getState());  // RUNNABLE

        synchronized (lock) {  // BLOCKED → RUNNABLE (락 획득)
            System.out.println("[" + id + "] Lock acquired, RUNNING");

            try {
                Thread.sleep(1000);  // TIMED_WAITING
            } catch (InterruptedException e) {}

            System.out.println("[" + id + "] Work done");
        }
    }
}

public class Main {
    public static void main(String[] args) throws InterruptedException {
        Object lock = new Object();

        Thread t1 = new Thread(new Worker(lock, 1));  // NEW
        Thread t2 = new Thread(new Worker(lock, 2));

        System.out.println("t1 state: " + t1.getState());  // NEW

        t1.start();  // RUNNABLE
        t2.start();

        System.out.println("t1 state: " + t1.getState());  // RUNNABLE or BLOCKED

        t1.join();  // WAITING (t1 종료 대기)
        t2.join();

        System.out.println("t1 state: " + t1.getState());  // TERMINATED
    }
}
```

---

## 📊 상태별 시간 분석

### 스레드 시간 분류

```
전체 실행 시간 = Running Time + Runnable Time + Blocked Time

- Running Time: 실제 CPU 사용 시간
- Runnable Time: CPU를 기다리는 시간 (스케줄링 대기)
- Blocked Time: I/O, Lock 등을 기다리는 시간
```

### 효율성 지표

```
CPU Utilization = Running Time / Total Time
Throughput = Completed Tasks / Total Time
```

**최적화 목표**:
- Runnable Time 감소 → 스레드 수 조정
- Blocked Time 감소 → 비동기 I/O, Lock-Free 자료구조

---

## ⚠️ 일반적인 문제

### 1. 과도한 Runnable 시간

**원인**: 스레드 수 > CPU 코어 수

**증상**: Context Switching 오버헤드 증가

**해결**: 스레드 풀 크기 조정 (코어 수의 1-2배)

### 2. 긴 Blocked 시간

**원인**: I/O 대기, Lock 경합

**증상**: 처리량 저하

**해결**:
- 비동기 I/O (async/await)
- Lock-Free 자료구조
- Lock 범위 최소화

### 3. Starvation (기아)

**원인**: 우선순위 낮은 스레드가 계속 대기

**증상**: 일부 스레드가 실행되지 않음

**해결**: Fair Lock, Priority Aging

---

## 🔗 다음 단계

- [컨텍스트 스위칭](./04-context-switching.md) - 상태 전이의 비용
- [동기화 기법](../02-synchronization/README.md) - BLOCKED 상태 관리
- [동시성 문제](../03-concurrency-problems/README.md) - Deadlock, Starvation

---

## 🔬 내부 메커니즘 심층 분석

### Linux 커널 스레드 상태

```c
// include/linux/sched.h
// 실제 Linux 커널에서 사용하는 상태 정의
#define TASK_RUNNING           0x0000  // 실행 중 또는 실행 가능
#define TASK_INTERRUPTIBLE     0x0001  // 대기 중, 시그널로 깨울 수 있음
#define TASK_UNINTERRUPTIBLE   0x0002  // 대기 중, 시그널 무시 (I/O 대기)
#define __TASK_STOPPED         0x0004  // 중지됨 (SIGSTOP)
#define __TASK_TRACED          0x0008  // 디버거에 의해 추적 중
#define TASK_PARKED            0x0040  // 파킹됨 (kthread)
#define TASK_DEAD              0x0080  // 종료됨, 리소스 해제 대기
#define TASK_WAKEKILL          0x0100  // SIGKILL로 깨울 수 있음
#define TASK_WAKING            0x0200  // 깨우는 중
#define TASK_NOLOAD            0x0400  // load average에 포함 안 됨
#define TASK_NEW               0x0800  // 새로 생성됨
```

```
Linux 상태 전이 다이어그램 (상세):

       fork()/clone()
            ↓
    ┌───────────────┐
    │  TASK_NEW     │ ← 방금 생성됨
    └───────────────┘
            ↓ wake_up_new_task()
    ┌───────────────┐
    │ TASK_RUNNING  │ ← Run Queue에 추가됨
    │  (runnable)   │
    └───────────────┘
            ↓ schedule() 선택
    ┌───────────────┐
    │ TASK_RUNNING  │ ← CPU에서 실행 중
    │  (running)    │
    └───────────────┘
            │
     ┌──────┴──────┬─────────────┬─────────────┐
     ↓             ↓             ↓             ↓
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│INTERRUP │  │UNINTERR │  │ STOPPED │  │  DEAD   │
│TIBLE    │  │UPTIBLE  │  │         │  │         │
│wait_    │  │I/O,     │  │SIGSTOP  │  │exit()   │
│event()  │  │mutex    │  │         │  │         │
└────┬────┘  └────┬────┘  └────┬────┘  └─────────┘
     │            │            │
     └──────┬─────┴────────────┘
            ↓ wake_up()
    ┌───────────────┐
    │ TASK_RUNNING  │
    │  (runnable)   │
    └───────────────┘
```

### Run Queue와 CFS 스케줄러

```c
// kernel/sched/sched.h
struct rq {
    raw_spinlock_t lock;       // Run Queue 락

    unsigned int nr_running;    // TASK_RUNNING 스레드 수
    u64 nr_switches;           // Context Switch 횟수

    struct cfs_rq cfs;         // CFS Run Queue (Red-Black Tree)
    struct rt_rq rt;           // Real-Time Run Queue
    struct dl_rq dl;           // Deadline Run Queue

    struct task_struct *curr;  // 현재 실행 중인 태스크
    struct task_struct *idle;  // Idle 태스크

    // CPU별 통계
    u64 clock;                 // 현재 시간
    u64 clock_task;            // 태스크 시간

    // Load tracking
    struct sched_avg avg;
};

// CFS Run Queue (완전 공정 스케줄러)
struct cfs_rq {
    struct rb_root_cached tasks_timeline;  // Red-Black Tree
    struct sched_entity *curr;             // 현재 실행 중
    u64 min_vruntime;                      // 최소 vruntime

    unsigned int nr_running;               // 실행 가능 태스크 수
};
```

```
CFS Red-Black Tree:

                    ┌─────┐
                    │vrt=5│ ← 루트
                    └──┬──┘
                 ┌─────┴─────┐
              ┌──┴──┐     ┌──┴──┐
              │vrt=3│     │vrt=8│
              └──┬──┘     └──┬──┘
            ┌────┴────┐      └────┐
         ┌──┴──┐   ┌──┴──┐    ┌──┴──┐
         │vrt=1│   │vrt=4│    │vrt=9│
         └─────┘   └─────┘    └─────┘
            ↑
       leftmost (다음 실행 대상)

vruntime 계산:
vruntime += delta_exec * (NICE_0_LOAD / weight)

nice 0 (기본): weight = 1024
nice -5:       weight = 3121  (더 많이 실행)
nice +5:       weight = 335   (덜 실행)
```

### Wait Queue 구현

```c
// include/linux/wait.h
struct wait_queue_head {
    spinlock_t lock;
    struct list_head head;
};

struct wait_queue_entry {
    unsigned int flags;
    void *private;              // 대기 중인 task_struct
    wait_queue_func_t func;     // 깨우기 함수
    struct list_head entry;
};

// 사용 예시 (커널 내부)
DECLARE_WAIT_QUEUE_HEAD(my_wait_queue);

// 대기 (TASK_INTERRUPTIBLE로 전환)
wait_event_interruptible(my_wait_queue, condition);

// 내부 동작:
// 1. current를 wait queue에 추가
// 2. set_current_state(TASK_INTERRUPTIBLE)
// 3. schedule() 호출 → 다른 태스크 실행
// 4. 깨어나면 condition 확인
// 5. condition false면 다시 대기

// 깨우기
wake_up(&my_wait_queue);       // 하나만 깨움
wake_up_all(&my_wait_queue);   // 모두 깨움
```

### /proc 파일시스템으로 상태 확인

```bash
# 스레드 상태 확인
$ cat /proc/[pid]/stat
# 3번째 필드가 상태
# R: Running
# S: Sleeping (TASK_INTERRUPTIBLE)
# D: Disk sleep (TASK_UNINTERRUPTIBLE) ← "unkillable"
# T: Stopped
# Z: Zombie

# 예시
$ cat /proc/1234/stat
1234 (myprogram) S 1233 1234 1234 ...
                 ^ 상태: Sleeping

# 스레드별 상태 (멀티스레드 프로그램)
$ ls /proc/1234/task/
1234  1235  1236  1237

$ cat /proc/1234/task/1235/stat
1235 (worker-1) R ...  # Running

$ cat /proc/1234/task/1236/stat
1236 (worker-2) D ...  # Disk sleep (I/O 대기)

# D 상태 원인 분석
$ cat /proc/1236/stack
[<0>] io_schedule+0x46/0x70
[<0>] blk_mq_get_tag+0x123/0x2a0
[<0>] __blk_mq_alloc_request+0x6d/0x150
...
```

### TASK_UNINTERRUPTIBLE (D 상태) 이해

```
TASK_UNINTERRUPTIBLE의 중요성:

┌────────────────────────────────────────────────────────┐
│ 상황: 디스크 I/O 진행 중                               │
│                                                        │
│ 만약 INTERRUPTIBLE이라면:                              │
│ 1. read() 시스템 콜 진행 중                           │
│ 2. 시그널 도착 (예: SIGTERM)                          │
│ 3. 시스템 콜 중단됨                                    │
│ 4. 하지만 하드웨어는 아직 DMA 전송 중!                │
│ 5. 데이터 불일치 → 파일시스템 손상 가능               │
│                                                        │
│ TASK_UNINTERRUPTIBLE 사용:                            │
│ 1. read() 시스템 콜 진행 중                           │
│ 2. 시그널 도착 → 무시됨                               │
│ 3. DMA 완료될 때까지 대기                             │
│ 4. 완료 후 시그널 처리                                │
│ 5. 데이터 일관성 보장                                  │
└────────────────────────────────────────────────────────┘

주의: D 상태 스레드는 kill -9로도 종료 불가!
해결: I/O 완료 대기 또는 시스템 재부팅
```

### ftrace로 스케줄러 추적

```bash
# ftrace 활성화
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_wakeup/enable

# 추적 시작
$ echo 1 > /sys/kernel/debug/tracing/tracing_on
$ ./my_program &
$ sleep 1
$ echo 0 > /sys/kernel/debug/tracing/tracing_on

# 결과 확인
$ cat /sys/kernel/debug/tracing/trace

# 출력 예시:
#           TASK-PID   CPU#  |  TIMESTAMP  FUNCTION
#              | |       |   |      |        |
         worker-1234  [001]  1234.567890: sched_switch:
                prev_comm=worker prev_pid=1234 prev_state=S
                ==> next_comm=worker-2 next_pid=1235

         worker-2-1235  [001]  1234.567900: sched_wakeup:
                comm=worker pid=1234 target_cpu=001

# prev_state 해석:
# R = TASK_RUNNING
# S = TASK_INTERRUPTIBLE
# D = TASK_UNINTERRUPTIBLE
# T = __TASK_STOPPED
# t = __TASK_TRACED
# X = TASK_DEAD
# Z = EXIT_ZOMBIE
```

---

## 📚 참고 자료

- "Operating Systems: Three Easy Pieces" - Chapter 7 (Scheduling)
- [Java Thread States](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.State.html)
- Linux Kernel Source: `kernel/sched/core.c`, `kernel/sched/fair.c`
- "Understanding the Linux Kernel" - Chapter 7 (Process Scheduling)

---

*스레드 생명주기를 이해하면 성능 병목 지점을 정확히 진단할 수 있습니다!*
