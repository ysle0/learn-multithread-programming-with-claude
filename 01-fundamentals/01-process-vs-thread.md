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

## 📖 참고 자료

- "Operating Systems: Three Easy Pieces" - Chapter 26 (Threads)
- "The Linux Programming Interface" - Chapter 28 (Processes)
- POSIX Threads Programming: https://computing.llnl.gov/tutorials/pthreads/

---

*프로세스와 스레드의 차이를 이해하는 것은 멀티스레드 프로그래밍의 첫 걸음입니다.*
