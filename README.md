# 멀티스레드 프로그래밍 학습 레포지토리

[![Language](https://img.shields.io/badge/Language-Korean-blue.svg)](README.md)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

**멀티스레드 프로그래밍을 표면적인 개념부터 가장 깊은 수준까지 체계적으로 학습할 수 있는 종합 가이드**

---

## 📌 개요

이 레포지토리는 동시성(Concurrency)과 병렬성(Parallelism) 프로그래밍을 처음 배우는 초보자부터, Lock-Free 자료구조와 게임 서버 아키텍처를 이해하려는 고급 개발자까지 모두를 위한 학습 자료입니다.

### 🎯 학습 목표

이 레포지토리를 완료하면 다음을 할 수 있습니다:

- ✅ 프로세스, 스레드, 동시성의 기초 개념 이해
- ✅ Mutex, Semaphore, Condition Variable 등의 동기화 기법 활용
- ✅ Race Condition, Deadlock 등의 동시성 문제 진단 및 해결
- ✅ Producer-Consumer, Thread Pool 등의 동시성 패턴 구현
- ✅ Lock-Free/Wait-Free 자료구조 이해 및 설계
- ✅ C++, C#, Go, JavaScript 등 여러 언어의 동시성 모델 비교
- ✅ Windows와 POSIX 플랫폼 차이 이해 및 크로스 플랫폼 개발
- ✅ 실시간 게임 서버와 MMO 아키텍처 설계
- ✅ 오픈소스 프로젝트(Folly, libcds, Nakama) 분석

### 🌟 이 레포지토리의 특징

| 특징 | 설명 |
|------|------|
| **체계적인 구성** | 기초부터 고급까지 단계별 학습 경로 |
| **풍부한 예시** | 모든 개념에 실제 동작하는 코드 예시 포함 |
| **다중 언어** | C++, C#, Go, JavaScript 비교 |
| **실전 응용** | 게임 서버, 웹 서버 등 실무 아키텍처 |
| **오픈소스 분석** | 실제 프로덕션 라이브러리 코드 분석 |
| **한글 설명** | 한국어로 작성된 상세한 설명 |

---

## 📂 디렉토리 구조

```
learn-multithread-programming-with-claude/
│
├── 01-fundamentals/                 ← 기초 개념
│   ├── 01-process-vs-thread.md
│   ├── 02-concurrency-vs-parallelism.md
│   ├── 03-thread-lifecycle.md
│   └── 04-context-switching.md
│
├── 02-synchronization/              ← 동기화 기법
│   ├── 01-mutex-lock.md
│   ├── 02-semaphore.md
│   ├── 03-condition-variable.md
│   ├── 04-atomic-operations.md
│   ├── 05-memory-barrier.md
│   └── 06-rwlock.md
│
├── 03-concurrency-problems/         ← 동시성 문제
│   ├── 01-race-condition.md
│   ├── 02-deadlock.md
│   ├── 03-livelock.md
│   ├── 04-starvation.md
│   └── 05-priority-inversion.md
│
├── 04-concurrency-patterns/         ← 동시성 패턴
│   ├── 01-producer-consumer.md
│   ├── 02-reader-writer.md
│   ├── 03-thread-pool.md
│   ├── 04-actor-model.md
│   ├── 05-future-promise.md
│   ├── 06-pipeline.md
│   └── 07-fan-out-fan-in.md
│
├── 05-lock-free-programming/        ← Lock-Free/Wait-Free
│   ├── 03-lock-free-stack.md (Treiber Stack)
│   ├── 04-lock-free-queue.md (Michael-Scott Queue)
│   ├── 05-lock-free-counter.md
│   └── 06-wait-free-programming.md
│
├── 06-language-implementations/     ← 언어별 구현
│   ├── cpp/                         (C++17/20)
│   ├── csharp/                      (.NET 6+)
│   ├── go/                          (Go 1.20+)
│   └── javascript/                  (ES2022+)
│
├── 07-game-server-applications/     ← 게임 서버 적용
│   ├── 01-realtime-game-server.md   (FPS, MOBA, MMO)
│   ├── 02-non-realtime-server.md
│   ├── 03-mmo-architecture.md
│   ├── 04-game-loop-threading.md
│   └── 05-networking-threading.md   (epoll, IOCP)
│
├── 08-open-source-analysis/         ← 오픈소스 분석
│   ├── 01-libcds.md                 (C++ Concurrent DS)
│   ├── 02-folly.md                  (Facebook Folly)
│   ├── 03-nakama.md                 (Game Server)
│   ├── 04-colyseus.md               (Multiplayer Framework)
│   └── 05-xsync.md                  (Go sync)
│
├── 09-appendix/                     ← 부록
│   ├── debugging-tools.md           (ThreadSanitizer, Valgrind)
│   ├── testing-strategies.md
│   ├── performance-tuning.md
│   └── references.md
│
└── 10-platform-differences/         ← 플랫폼 차이 (Windows vs POSIX)
    ├── 01-thread-creation.md        (CreateThread vs pthread)
    ├── 02-synchronization-primitives.md
    ├── 03-thread-local-storage.md
    ├── 04-ipc.md
    ├── 05-scheduling.md
    ├── 06-error-handling.md
    ├── 07-portability-layer.md
    └── 08-performance-comparison.md
```

---

## 🎓 학습 경로

### 초급 (필수 기초)

**목표**: 동시성 프로그래밍의 기본 개념 이해

```
1주차: 기초 개념
├── 01-fundamentals/
│   ├── 프로세스 vs 스레드
│   ├── 동시성 vs 병렬성
│   ├── 스레드 생명주기
│   └── 컨텍스트 스위칭

2주차: 동기화 기법
└── 02-synchronization/
    ├── Mutex / Lock
    ├── Semaphore
    └── Condition Variable
```

**예상 시간**: 2-3주
**평가 기준**: 간단한 Producer-Consumer 구현 가능

---

### 중급 (실무 활용)

**목표**: 동시성 문제 해결 및 패턴 적용

```
3주차: 동시성 문제
├── 03-concurrency-problems/
│   ├── Race Condition 탐지 및 해결
│   ├── Deadlock 예방 및 복구
│   └── Starvation 방지

4-5주차: 동시성 패턴
└── 04-concurrency-patterns/
    ├── Producer-Consumer
    ├── Thread Pool
    ├── Future/Promise
    └── Pipeline
```

**예상 시간**: 3-4주
**평가 기준**: 멀티스레드 웹 서버 구현 가능

---

### 고급 (Lock-Free 및 게임 서버)

**목표**: Lock-Free 자료구조 이해 및 고성능 서버 아키텍처 설계

```
6-7주차: Lock-Free 프로그래밍
├── 05-lock-free-programming/
│   ├── CAS 연산 및 Memory Ordering
│   ├── Treiber Stack
│   ├── Michael-Scott Queue
│   ├── ABA Problem 해결
│   └── Wait-Free 프로그래밍

8-9주차: 게임 서버 아키텍처
└── 07-game-server-applications/
    ├── 실시간 게임 서버 (FPS, MOBA)
    ├── MMO 아키텍처 (Sharding, Partitioning)
    ├── Game Loop Threading
    └── Networking I/O (epoll, IOCP)
```

**예상 시간**: 4-6주
**평가 기준**: Lock-Free Queue 구현 및 간단한 게임 서버 작성 가능

---

### 전문가 (언어별 구현 및 오픈소스 분석)

**목표**: 다양한 언어의 동시성 모델 이해 및 프로덕션 코드 분석

```
10주차: 언어별 비교
└── 06-language-implementations/
    ├── C++: std::thread, std::atomic
    ├── C#: Task, async/await
    ├── Go: Goroutine, Channel
    └── JavaScript: Event Loop, Worker

11-12주차: 오픈소스 분석
└── 08-open-source-analysis/
    ├── libcds (Lock-Free 라이브러리)
    ├── Folly (Facebook 고성능 라이브러리)
    ├── Nakama (게임 서버)
    └── xsync (Go 동시성)
```

**예상 시간**: 3-4주
**평가 기준**: 오픈소스 프로젝트 기여 가능

---

## 📚 섹션별 상세 가이드

### [01. 기초 개념 (Fundamentals)](./01-fundamentals/README.md)

멀티스레드 프로그래밍의 토대가 되는 핵심 개념을 학습합니다.

| 문서 | 핵심 내용 | 난이도 |
|------|----------|--------|
| [프로세스 vs 스레드](./01-fundamentals/01-process-vs-thread.md) | 메모리 구조, 자원 공유, IPC | ⭐ |
| [동시성 vs 병렬성](./01-fundamentals/02-concurrency-vs-parallelism.md) | Amdahl's Law, CPU-bound vs I/O-bound | ⭐ |
| [스레드 생명주기](./01-fundamentals/03-thread-lifecycle.md) | 상태 전이, 스케줄링 알고리즘 | ⭐⭐ |
| [컨텍스트 스위칭](./01-fundamentals/04-context-switching.md) | 오버헤드, 캐시/TLB 영향 | ⭐⭐ |

**학습 목표**: "왜 멀티스레드가 필요한가?"에 답할 수 있어야 합니다.

---

### [02. 동기화 기법 (Synchronization)](./02-synchronization/README.md)

스레드 간 안전한 데이터 공유를 위한 동기화 메커니즘을 학습합니다.

| 문서 | 핵심 내용 | 난이도 |
|------|----------|--------|
| [Mutex / Lock](./02-synchronization/01-mutex-lock.md) | Critical Section, RAII, Spinlock | ⭐⭐ |
| [Semaphore](./02-synchronization/02-semaphore.md) | Counting Semaphore, Resource Pool | ⭐⭐ |
| [Condition Variable](./02-synchronization/03-condition-variable.md) | wait/notify, Spurious Wakeup | ⭐⭐⭐ |
| [Atomic Operations](./02-synchronization/04-atomic-operations.md) | CAS, Memory Ordering 기초 | ⭐⭐⭐ |
| [Memory Barrier](./02-synchronization/05-memory-barrier.md) | Acquire-Release, Sequential Consistency | ⭐⭐⭐⭐ |
| [Reader-Writer Lock](./02-synchronization/06-rwlock.md) | std::shared_mutex, Fair RWLock | ⭐⭐ |

**학습 목표**: 적절한 동기화 도구를 선택하고 사용할 수 있어야 합니다.

---

### [03. 동시성 문제 (Concurrency Problems)](./03-concurrency-problems/README.md)

멀티스레드 환경에서 발생하는 전형적인 문제들을 학습합니다.

| 문서 | 핵심 내용 | 난이도 |
|------|----------|--------|
| [Race Condition](./03-concurrency-problems/01-race-condition.md) | Read-Modify-Write, ThreadSanitizer | ⭐⭐ |
| [Deadlock](./03-concurrency-problems/02-deadlock.md) | 4가지 조건, Banker's Algorithm | ⭐⭐⭐ |
| [Livelock](./03-concurrency-problems/03-livelock.md) | Deadlock과의 차이, Backoff 전략 | ⭐⭐⭐ |
| [Starvation](./03-concurrency-problems/04-starvation.md) | Fairness, Priority Aging | ⭐⭐ |
| [Priority Inversion](./03-concurrency-problems/05-priority-inversion.md) | Mars Pathfinder 사례, Priority Inheritance | ⭐⭐⭐⭐ |

**학습 목표**: 동시성 버그를 진단하고 해결할 수 있어야 합니다.

---

### [04. 동시성 패턴 (Concurrency Patterns)](./04-concurrency-patterns/README.md)

실무에서 자주 사용되는 검증된 동시성 패턴을 학습합니다.

| 문서 | 핵심 내용 | 사용 사례 | 난이도 |
|------|----------|----------|--------|
| [Producer-Consumer](./04-concurrency-patterns/01-producer-consumer.md) | Bounded Buffer, Backpressure | 작업 큐, 로그 처리 | ⭐⭐ |
| [Reader-Writer](./04-concurrency-patterns/02-reader-writer.md) | Fair Lock, Writer Starvation | 캐시, 설정 관리 | ⭐⭐ |
| [Thread Pool](./04-concurrency-patterns/03-thread-pool.md) | Work Stealing, Dynamic Sizing | 웹 서버, 배치 처리 | ⭐⭐⭐ |
| [Actor Model](./04-concurrency-patterns/04-actor-model.md) | Message Passing, Supervision | 분산 시스템 | ⭐⭐⭐⭐ |
| [Future/Promise](./04-concurrency-patterns/05-future-promise.md) | Continuation, Composition | 비동기 API | ⭐⭐⭐ |
| [Pipeline](./04-concurrency-patterns/06-pipeline.md) | Stage 연결, Backpressure | 데이터 처리 | ⭐⭐⭐ |
| [Fan-Out/Fan-In](./04-concurrency-patterns/07-fan-out-fan-in.md) | 병렬 분산, 결과 수집 | MapReduce | ⭐⭐⭐ |

**학습 목표**: 패턴을 적용하여 확장 가능한 시스템을 설계할 수 있어야 합니다.

---

### [05. Lock-Free / Wait-Free 프로그래밍](./05-lock-free-programming/README.md)

락 없이 동시성을 제어하는 고급 기법을 학습합니다.

| 문서 | 핵심 내용 | 난이도 |
|------|----------|--------|
| [Lock-Free Stack](./05-lock-free-programming/03-lock-free-stack.md) | Treiber Stack, CAS 루프 | ⭐⭐⭐⭐ |
| [Lock-Free Queue](./05-lock-free-programming/04-lock-free-queue.md) | Michael-Scott Queue, Dummy 노드 | ⭐⭐⭐⭐⭐ |
| [Lock-Free Counter](./05-lock-free-programming/05-lock-free-counter.md) | fetch_add, CAS vs Fetch-Add | ⭐⭐⭐ |
| [Wait-Free Programming](./05-lock-free-programming/06-wait-free-programming.md) | Progress Guarantee, SPSC Queue | ⭐⭐⭐⭐⭐ |

**학습 목표**: Lock-Free 알고리즘의 원리를 이해하고 간단한 자료구조를 구현할 수 있어야 합니다.

**⚠️ 경고**: Lock-Free 프로그래밍은 매우 어렵습니다. 실무에서는 검증된 라이브러리(Folly, libcds) 사용을 권장합니다.

---

### [06. 언어별 구현 (Language Implementations)](./06-language-implementations/README.md)

C++, C#, Go, JavaScript의 동시성 모델을 비교 학습합니다.

#### [C++ (Modern C++17/20)](./06-language-implementations/cpp/README.md)
- std::thread, std::mutex, std::atomic
- RAII 기반 락 관리
- std::async, std::future

#### [C# (.NET 6+)](./06-language-implementations/csharp/README.md)
- Task vs Thread
- async/await 패턴
- Concurrent Collections

#### [Go (Goroutine 기반)](./06-language-implementations/go/README.md)
- Goroutine과 Channel
- CSP 모델: "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라"
- Context를 이용한 취소 처리

#### [JavaScript (Event Loop)](./06-language-implementations/javascript/README.md)
- Single-threaded Event Loop
- async/await (Promise 기반)
- Web Workers, Worker Threads

**학습 목표**: 언어별 철학과 최적의 패턴을 이해해야 합니다.

---

### [07. 게임 서버 적용 (Game Server Applications)](./07-game-server-applications/README.md)

멀티스레드 프로그래밍을 게임 서버에 적용하는 방법을 학습합니다.

| 문서 | 핵심 내용 | 예시 |
|------|----------|------|
| [실시간 게임 서버](./07-game-server-applications/01-realtime-game-server.md) | 60-128 Hz Tick Rate, Lag Compensation | FPS, MOBA, MMO |
| [비실시간 서버](./07-game-server-applications/02-non-realtime-server.md) | Turn-Based, REST API | 카드 게임, 전략 게임 |
| [MMO 아키텍처](./07-game-server-applications/03-mmo-architecture.md) | Sharding, Partitioning, Zone Server | WoW, EVE Online |
| [Game Loop Threading](./07-game-server-applications/04-game-loop-threading.md) | Fixed Timestep, Job System | Unity, Unreal |
| [Networking Threading](./07-game-server-applications/05-networking-threading.md) | epoll, IOCP, 비동기 I/O | 네트워크 계층 |

**학습 목표**: 게임 서버 아키텍처를 설계하고 성능 최적화를 할 수 있어야 합니다.

---

### [08. 오픈소스 분석 (Open Source Analysis)](./08-open-source-analysis/README.md)

실제 프로덕션 라이브러리의 코드를 분석하여 실무 패턴을 학습합니다.

| 프로젝트 | 언어 | 핵심 기술 |
|----------|------|----------|
| [libcds](./08-open-source-analysis/01-libcds.md) | C++ | Hazard Pointers, Lock-Free DS |
| [Folly](./08-open-source-analysis/02-folly.md) | C++ | MPMCQueue, Future/Promise |
| [Nakama](./08-open-source-analysis/03-nakama.md) | Go | Goroutine 기반 게임 서버 |
| [Colyseus](./08-open-source-analysis/04-colyseus.md) | Node.js | 상태 동기화 |
| [xsync](./08-open-source-analysis/05-xsync.md) | Go | Generics 기반 동시성 DS |

**학습 목표**: 오픈소스 코드를 읽고 이해하여 자신의 프로젝트에 적용할 수 있어야 합니다.

---

### [09. 부록 (Appendix)](./09-appendix/)

실전 개발에 필요한 도구와 전략을 학습합니다.

| 문서 | 내용 |
|------|------|
| [디버깅 도구](./09-appendix/debugging-tools.md) | ThreadSanitizer, Valgrind, GDB |
| [테스트 전략](./09-appendix/testing-strategies.md) | Stress Testing, Property-Based Testing |
| [성능 튜닝](./09-appendix/performance-tuning.md) | Profiling, False Sharing, Lock Contention |
| [참고 자료](./09-appendix/references.md) | 책, 논문, 강의, 블로그 |

---

### [10. 플랫폼 차이 (Windows vs POSIX)](./10-platform-differences/README.md)

크로스 플랫폼 멀티스레드 프로그래밍을 위한 플랫폼별 차이점을 학습합니다.

| 문서 | 핵심 내용 | 난이도 |
|------|----------|--------|
| [스레드 생성](./10-platform-differences/01-thread-creation.md) | CreateThread vs pthread_create | ⭐⭐ |
| [동기화 프리미티브](./10-platform-differences/02-synchronization-primitives.md) | CRITICAL_SECTION vs pthread_mutex_t | ⭐⭐⭐ |
| [Thread Local Storage](./10-platform-differences/03-thread-local-storage.md) | TlsAlloc vs pthread_key_create | ⭐⭐⭐ |
| [IPC](./10-platform-differences/04-ipc.md) | Named Objects vs POSIX IPC | ⭐⭐⭐ |
| [스케줄링](./10-platform-differences/05-scheduling.md) | Priority Classes vs Nice Values | ⭐⭐⭐⭐ |
| [에러 처리](./10-platform-differences/06-error-handling.md) | GetLastError vs errno | ⭐⭐ |
| [이식성 레이어](./10-platform-differences/07-portability-layer.md) | C++11 std::thread, 조건부 컴파일 | ⭐⭐⭐ |
| [성능 비교](./10-platform-differences/08-performance-comparison.md) | 플랫폼별 벤치마크 | ⭐⭐⭐⭐ |

**학습 목표**: Windows와 POSIX의 차이를 이해하고 이식 가능한 코드를 작성할 수 있어야 합니다.

**💡 권장**: C++11 `std::thread`를 사용하면 대부분의 플랫폼 차이를 걱정하지 않아도 됩니다.

---

## 📊 난이도 가이드

| 레벨 | 설명 | 예시 |
|------|------|------|
| ⭐ | 기초 개념 | 프로세스 vs 스레드 |
| ⭐⭐ | 기본 도구 사용 | Mutex, Semaphore |
| ⭐⭐⭐ | 복잡한 패턴 | Thread Pool, Deadlock |
| ⭐⭐⭐⭐ | 고급 기법 | Lock-Free Stack, Priority Inversion |
| ⭐⭐⭐⭐⭐ | 전문가 수준 | Michael-Scott Queue, Wait-Free |

---

## 🛠️ 개발 환경 설정

### C++

```bash
# GCC/Clang with C++17 support
g++ -std=c++17 -pthread -O2 -Wall example.cpp -o example

# ThreadSanitizer 활성화
g++ -std=c++17 -pthread -fsanitize=thread -g example.cpp -o example
```

### C#

```bash
# .NET 6+ SDK 필요
dotnet new console -n ConcurrencyExample
dotnet run
```

### Go

```bash
# Go 1.20+ 권장
go mod init example
go run main.go

# Race Detector
go run -race main.go
```

### JavaScript/TypeScript

```bash
# Node.js 18+ 권장
npm init -y
npm install --save-dev typescript @types/node

# 실행
node example.js
# 또는
ts-node example.ts
```

---

## 📖 추천 학습 자료

### 서적

| 제목 | 저자 | 난이도 | 언어 |
|------|------|--------|------|
| "C++ Concurrency in Action" | Anthony Williams | ⭐⭐⭐⭐ | C++ |
| "Java Concurrency in Practice" | Brian Goetz | ⭐⭐⭐ | Java |
| "Concurrency in Go" | Katherine Cox-Buday | ⭐⭐⭐ | Go |
| "The Art of Multiprocessor Programming" | Herlihy & Shavit | ⭐⭐⭐⭐⭐ | 이론 |
| "Operating Systems: Three Easy Pieces" | Arpaci-Dusseau | ⭐⭐ | OS |

### 온라인 강의

- [MIT 6.824: Distributed Systems](https://pdos.csail.mit.edu/6.824/)
- [Coursera: Parallel Programming in Scala](https://www.coursera.org/learn/scala-parallel-programming)
- [Udemy: Multithreading and Concurrency in C++](https://www.udemy.com/course/multithreading-and-concurrency-in-cpp/)

### 블로그/웹사이트

- [Preshing on Programming](https://preshing.com/) - Lock-Free 프로그래밍
- [1024cores](http://www.1024cores.net/home) - Dmitry Vyukov의 동시성 기술
- [Mechanical Sympathy](https://mechanical-sympathy.blogspot.com/) - Martin Thompson

---

## 🤝 기여 가이드

이 레포지토리는 학습 목적으로 공개되어 있으며, 기여를 환영합니다!

### 기여 방법

1. **오류 수정**: 오타, 코드 버그 등 발견 시 Issue 또는 PR
2. **예시 추가**: 새로운 언어나 패턴의 예시 코드
3. **번역**: 영어 버전 작성 (현재 한글만 제공)
4. **실전 사례**: 실무에서 적용한 사례 공유

### 문서 작성 가이드

- 간결하고 명확한 설명
- 동작하는 코드 예시 포함
- ASCII 다이어그램으로 시각화
- 관련 문서로의 링크 제공

---

## 📜 라이센스

이 프로젝트는 MIT 라이센스 하에 공개됩니다. 자유롭게 사용, 수정, 배포할 수 있습니다.

---

## 🙏 감사의 말

이 레포지토리는 다음 자료들을 참고하여 작성되었습니다:

- "C++ Concurrency in Action" - Anthony Williams
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Operating Systems: Three Easy Pieces" - Arpaci-Dusseau
- Preshing on Programming
- 1024cores
- Go Blog의 Concurrency Patterns

---

## 💬 연락처

질문, 제안, 피드백은 GitHub Issues를 통해 부탁드립니다.

**Happy Learning! 멀티스레드 프로그래밍 마스터를 향해! 🚀**

---

*최종 업데이트: 2026-01-20*
