# 멀티스레드 프로그래밍 기초

## 📌 개요

이 섹션에서는 멀티스레드 프로그래밍을 이해하기 위한 **필수 기초 개념**을 다룹니다. 동시성 프로그래밍의 표면적인 부분부터 시작하여 운영체제 레벨의 깊은 내용까지 체계적으로 학습합니다.

---

## 🎯 학습 목표

이 섹션을 완료하면 다음을 이해할 수 있습니다:

1. ✅ 프로세스와 스레드의 차이와 각각의 장단점
2. ✅ 동시성과 병렬성의 개념적 차이
3. ✅ 스레드의 생명주기와 상태 전이
4. ✅ 컨텍스트 스위칭의 원리와 성능 영향
5. ✅ Thread Local Storage의 내부 동작과 활용 방법

---

## 📚 문서 목록

### [01. 프로세스 vs 스레드](./01-process-vs-thread.md)
**핵심 개념**: 프로세스는 독립적인 메모리 공간을 가지지만, 스레드는 같은 프로세스 내에서 메모리를 공유합니다.

**다루는 내용**:
- 프로세스와 스레드의 메모리 구조 비교
- 자원 공유와 격리
- IPC vs 스레드 간 통신
- 커널 스레드 vs 유저 스레드

**왜 중요한가**: 멀티스레드 vs 멀티프로세스 아키텍처를 선택하는 기준이 됩니다.

---

### [02. 동시성 vs 병렬성](./02-concurrency-vs-parallelism.md)
**핵심 개념**: 동시성(Concurrency)은 "동시에 처리하는 것처럼 보이는 것", 병렬성(Parallelism)은 "실제로 동시에 실행하는 것"입니다.

**다루는 내용**:
- 동시성과 병렬성의 정의와 차이
- 단일 코어에서의 동시성
- 멀티 코어에서의 병렬성
- Amdahl's Law와 Gustafson's Law
- CPU-bound vs I/O-bound 작업

**왜 중요한가**: 시스템 아키텍처를 설계할 때 성능 최적화의 방향을 결정합니다.

---

### [03. 스레드 생명주기](./03-thread-lifecycle.md)
**핵심 개념**: 스레드는 생성, 실행, 대기, 종료 등의 상태를 거치며, 상태 전이는 운영체제 스케줄러가 관리합니다.

**다루는 내용**:
- 스레드 상태: New, Runnable, Running, Blocked, Terminated
- 상태 전이 다이어그램
- 스케줄링 알고리즘 개요 (FCFS, Round-Robin, Priority)
- 선점형 vs 비선점형 스케줄링

**왜 중요한가**: 스레드가 블로킹되는 이유를 이해하고, 성능 문제를 진단할 수 있습니다.

---

### [04. 컨텍스트 스위칭](./04-context-switching.md)
**핵심 개념**: 컨텍스트 스위칭은 CPU가 한 스레드에서 다른 스레드로 전환할 때 발생하며, 레지스터 저장/복원 등의 오버헤드가 있습니다.

**다루는 내용**:
- 컨텍스트 스위칭 과정
- 레지스터 저장/복원
- CPU 캐시와 TLB의 영향
- 컨텍스트 스위칭 측정 방법
- 오버헤드 최소화 전략

**왜 중요한가**: 스레드를 무분별하게 생성하면 컨텍스트 스위칭 오버헤드로 성능이 저하될 수 있습니다.

---

### [05. Thread Local Storage (TLS)](./05-thread-local-storage.md)
**핵심 개념**: TLS는 각 스레드가 고유한 데이터 복사본을 가질 수 있게 하여, 동기화 없이 스레드 안전한 코드를 작성할 수 있게 합니다.

**다루는 내용**:
- TLS의 개념과 필요성
- 내부 동작 원리 (x86-64 FS 레지스터, TCB, DTV)
- ELF TLS 모델 (Local Exec, Initial Exec, Local Dynamic, General Dynamic)
- 정적 TLS vs 동적 TLS
- Use Cases (Thread-Local Pool, errno, Cache, Random Generator)
- 성능 분석 및 Best Practices

**왜 중요한가**: Thread-Local Memory Pool, TCMalloc 등 고성능 메모리 할당기의 핵심 기술이며, errno 같은 스레드 안전한 전역 상태 관리에 필수적입니다.

---

## 🔍 핵심 개념 요약

### 프로세스 vs 스레드

| 특성 | 프로세스 | 스레드 |
|------|----------|--------|
| **메모리** | 독립적 (격리) | 공유 (Heap, Data 영역) |
| **생성 비용** | 높음 (~1-2ms) | 낮음 (~0.1ms) |
| **통신** | IPC (복잡) | 공유 메모리 (간단) |
| **안정성** | 높음 (크래시 격리) | 낮음 (한 스레드 크래시 시 전체 영향) |
| **사용 사례** | 독립된 서비스, 샌드박스 | 동일 애플리케이션 내 병렬 처리 |

### 동시성 vs 병렬성

```
동시성 (Concurrency):
Time ─────────────────────────────>
CPU:  [A][B][A][C][B][A][C]...
      (여러 작업을 번갈아가며 처리)

병렬성 (Parallelism):
Time ─────────────────────────────>
CPU1: [A][A][A][A][A]...
CPU2: [B][B][B][B][B]...
CPU3: [C][C][C][C][C]...
      (여러 작업을 동시에 처리)
```

**중요**: 단일 코어에서도 동시성은 가능하지만, 병렬성은 멀티 코어가 필수입니다.

### 스레드 상태 전이

```
      create()
         ↓
    ┌────────┐
    │  New   │
    └────────┘
         ↓ start()
    ┌────────────┐  ← yield() ← ┌─────────┐
    │  Runnable  │ ────────────→│ Running │
    └────────────┘   schedule    └─────────┘
         ↑                            │
         │                            │ I/O, Lock 대기
         │                            ↓
         │                       ┌─────────┐
         └───── I/O 완료, ────── │ Blocked │
                 Lock 획득        └─────────┘
                                      │
                                      │ terminate()
                                      ↓
                                 ┌───────────┐
                                 │Terminated │
                                 └───────────┘
```

### 컨텍스트 스위칭 비용

| 항목 | 시간 |
|------|------|
| **직접 비용** | 1-10 μs (레지스터 저장/복원) |
| **간접 비용** | 10-100 μs (캐시 미스, TLB flush) |
| **총 비용** | ~50-100 μs (시스템에 따라 다름) |

**교훈**: 스레드가 많을수록 좋은 것이 아니라, 적절한 수를 유지해야 합니다.

---

## 🎓 학습 경로

```
1. 프로세스 vs 스레드 (필수)
   ↓
2. 동시성 vs 병렬성 (필수)
   ↓
3. 스레드 생명주기 (필수)
   ↓
4. 컨텍스트 스위칭 (필수)
   ↓
5. Thread Local Storage (권장)
   ↓
다음 섹션: 02-synchronization/
```

**권장**: 순서대로 학습하되, 각 개념을 완벽히 이해한 후 다음으로 넘어가세요.

---

## 💡 실전 적용 팁

### 1. 스레드 vs 프로세스 선택 기준

**스레드 사용**:
- ✅ 같은 데이터를 공유해야 하는 경우
- ✅ 빠른 생성/종료가 필요한 경우
- ✅ 메모리 효율이 중요한 경우

**프로세스 사용**:
- ✅ 안정성이 최우선인 경우 (한 작업 실패가 전체에 영향 주면 안됨)
- ✅ 보안/격리가 필요한 경우 (브라우저 탭, 샌드박스)
- ✅ 서로 다른 권한이 필요한 경우

### 2. 최적 스레드 수 결정

```
CPU-bound 작업: 스레드 수 = CPU 코어 수
I/O-bound 작업: 스레드 수 = CPU 코어 수 × (1 + I/O 대기 시간 / CPU 시간)
```

**예시**:
- CPU 코어 4개
- I/O 대기 시간 80%, CPU 시간 20%
- 최적 스레드 수 = 4 × (1 + 80/20) = 4 × 5 = 20

### 3. 컨텍스트 스위칭 최소화

- ✅ 스레드 수를 적절히 유지 (코어 수의 1-2배)
- ✅ 스레드 풀 사용 (매번 생성/종료 방지)
- ✅ 락 경합 최소화 (블로킹 시간 감소)
- ✅ I/O-bound는 비동기 I/O 고려

---

## 📊 언어별 기본 예시

### C++ (std::thread)

```cpp
#include <thread>
#include <iostream>

void worker(int id) {
    std::cout << "Thread " << id << " running\n";
}

int main() {
    std::thread t1(worker, 1);
    std::thread t2(worker, 2);

    t1.join();  // 스레드 종료 대기
    t2.join();
    return 0;
}
```

### C# (Task)

```csharp
using System;
using System.Threading.Tasks;

class Program {
    static async Task Main() {
        var t1 = Task.Run(() => Console.WriteLine("Task 1"));
        var t2 = Task.Run(() => Console.WriteLine("Task 2"));

        await Task.WhenAll(t1, t2);
    }
}
```

### Go (Goroutine)

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    wg.Add(2)
    go func() {
        defer wg.Done()
        fmt.Println("Goroutine 1")
    }()
    go func() {
        defer wg.Done()
        fmt.Println("Goroutine 2")
    }()

    wg.Wait()
}
```

---

## 🔗 다음 단계

기초 개념을 이해했다면:

1. [동기화 기법](../02-synchronization/README.md) - 스레드 간 안전한 데이터 공유
2. [동시성 문제](../03-concurrency-problems/README.md) - Race Condition, Deadlock 등
3. [동시성 패턴](../04-concurrency-patterns/README.md) - 실전 패턴

---

## 📚 참고 자료

### 온라인
- [Operating Systems: Three Easy Pieces - Threads](https://pages.cs.wisc.edu/~remzi/OSTEP/)
- [Concurrent vs Parallel (Rob Pike)](https://go.dev/blog/waza-talk)

### 서적
- "Operating System Concepts" - Silberschatz (공룡책)
- "Modern Operating Systems" - Andrew Tanenbaum

---

*이 섹션은 멀티스레드 프로그래밍의 탄탄한 기초를 제공합니다. 서두르지 말고 천천히 이해하세요!*
