# 멀티스레드 프로그래밍 학습 레포지토리 계획

## 📌 개요

이 레포지토리는 멀티스레드 프로그래밍을 **표면적인 개념부터 가장 깊은 수준까지** 체계적으로 학습할 수 있도록 구성됩니다.

각 섹션은 **간결한 요약**으로 시작하고, 심화 내용은 별도 마크다운 링크로 연결됩니다.
모든 개념에는 **이해를 돕는 예시 코드**가 포함됩니다.

---

## 📂 디렉토리 구조

```
learn-multithread-programming-with-claude/
├── PLAN.md                          # 이 문서
├── README.md                        # 메인 진입점 (목차)
│
├── 01-fundamentals/                 # 기초 개념
│   ├── README.md                    # 개요
│   ├── 01-process-vs-thread.md      # 프로세스와 스레드 차이
│   ├── 02-concurrency-vs-parallelism.md  # 동시성 vs 병렬성
│   ├── 03-thread-lifecycle.md       # 스레드 생명주기
│   ├── 04-context-switching.md      # 컨텍스트 스위칭
│   └── examples/                    # 예시 코드
│
├── 02-synchronization/              # 동기화 기법
│   ├── README.md
│   ├── 01-mutex-lock.md             # Mutex / Lock
│   ├── 02-semaphore.md              # Semaphore
│   ├── 03-condition-variable.md     # Condition Variable
│   ├── 04-atomic-operations.md      # Atomic Operations
│   ├── 05-memory-barrier.md         # Memory Barrier / Fence
│   ├── 06-rwlock.md                 # Reader-Writer Lock
│   └── examples/
│
├── 03-concurrency-problems/         # 동시성 문제
│   ├── README.md
│   ├── 01-race-condition.md         # Race Condition
│   ├── 02-deadlock.md               # Deadlock
│   ├── 03-livelock.md               # Livelock
│   ├── 04-starvation.md             # Starvation
│   ├── 05-priority-inversion.md     # Priority Inversion
│   └── examples/
│
├── 04-concurrency-patterns/         # 동시성 패턴
│   ├── README.md
│   ├── 01-producer-consumer.md      # Producer-Consumer
│   ├── 02-reader-writer.md          # Reader-Writer
│   ├── 03-thread-pool.md            # Thread Pool
│   ├── 04-actor-model.md            # Actor Model
│   ├── 05-future-promise.md         # Future / Promise
│   ├── 06-pipeline.md               # Pipeline Pattern
│   ├── 07-fan-out-fan-in.md         # Fan-Out / Fan-In
│   └── examples/
│
├── 05-lock-free-programming/        # Lock-Free / Wait-Free 프로그래밍
│   ├── README.md
│   ├── 01-cas-operation.md          # Compare-And-Swap (CAS)
│   ├── 02-memory-ordering.md        # Memory Ordering
│   ├── 03-lock-free-queue.md        # Lock-Free Queue
│   ├── 04-lock-free-stack.md        # Lock-Free Stack
│   ├── 05-aba-problem.md            # ABA Problem
│   ├── 06-hazard-pointers.md        # Hazard Pointers
│   └── examples/
│
├── 06-language-implementations/     # 언어별 구현
│   ├── README.md                    # 언어 비교 개요
│   ├── cpp/                         # C++
│   │   ├── README.md
│   │   ├── 01-std-thread.md         # std::thread
│   │   ├── 02-mutex-lock-guard.md   # std::mutex, lock_guard
│   │   ├── 03-atomic.md             # std::atomic
│   │   ├── 04-condition-variable.md # std::condition_variable
│   │   ├── 05-async-future.md       # std::async, std::future
│   │   └── examples/
│   │
│   ├── csharp/                      # C#
│   │   ├── README.md
│   │   ├── 01-thread-task.md        # Thread vs Task
│   │   ├── 02-async-await.md        # async/await
│   │   ├── 03-lock-monitor.md       # lock, Monitor
│   │   ├── 04-semaphore-slim.md     # SemaphoreSlim
│   │   ├── 05-concurrent-collections.md  # Concurrent Collections
│   │   └── examples/
│   │
│   ├── go/                          # Go
│   │   ├── README.md
│   │   ├── 01-goroutine.md          # Goroutine
│   │   ├── 02-channel.md            # Channel
│   │   ├── 03-select.md             # Select
│   │   ├── 04-sync-package.md       # sync 패키지
│   │   ├── 05-context.md            # Context
│   │   └── examples/
│   │
│   └── javascript/                  # JavaScript
│       ├── README.md
│       ├── 01-event-loop.md         # Event Loop
│       ├── 02-async-await.md        # async/await
│       ├── 03-web-workers.md        # Web Workers
│       ├── 04-worker-threads.md     # Node.js Worker Threads
│       ├── 05-shared-array-buffer.md  # SharedArrayBuffer
│       └── examples/
│
├── 07-game-server-applications/     # 게임 서버 적용
│   ├── README.md
│   ├── 01-realtime-game-server.md   # 실시간 게임서버 적합성
│   ├── 02-non-realtime-server.md    # 비실시간 서버 (웹서버) 적합성
│   ├── 03-mmo-architecture.md       # MMO 아키텍처
│   ├── 04-game-loop-threading.md    # Game Loop와 스레딩
│   ├── 05-networking-threading.md   # 네트워킹과 멀티스레딩
│   └── examples/
│
├── 08-open-source-analysis/         # 오픈소스 분석
│   ├── README.md
│   ├── 01-libcds.md                 # libcds (C++ Concurrent Data Structures)
│   ├── 02-folly.md                  # Facebook Folly
│   ├── 03-nakama.md                 # Nakama Game Server
│   ├── 04-colyseus.md               # Colyseus
│   ├── 05-xsync.md                  # xsync (Go)
│   └── 06-recommended-repos.md      # 추천 레포지토리 목록
│
├── 09-appendix/                     # 부록
│   ├── debugging-tools.md           # 디버깅 도구
│   ├── testing-strategies.md        # 테스트 전략
│   ├── performance-tuning.md        # 성능 튜닝
│   └── references.md                # 참고 자료 링크
│
└── 10-platform-differences/         # 플랫폼 차이 (Windows vs POSIX)
    ├── README.md
    ├── 01-thread-creation.md        # CreateThread vs pthread_create
    ├── 02-synchronization-primitives.md  # Mutex, Event, Semaphore
    ├── 03-thread-local-storage.md   # TLS 비교
    ├── 04-ipc.md                    # 프로세스 간 통신
    ├── 05-scheduling.md             # 스케줄링 및 우선순위
    ├── 06-error-handling.md         # GetLastError vs errno
    ├── 07-portability-layer.md      # 크로스 플랫폼 추상화
    ├── 08-performance-comparison.md # 성능 벤치마크
    └── examples/
```

---

## 📋 섹션별 상세 계획

### 1. 기초 개념 (01-fundamentals/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| 프로세스 vs 스레드 | 메모리 구조, 자원 공유 차이 | 커널 스레드 vs 유저 스레드 |
| 동시성 vs 병렬성 | 개념 차이, 언제 어떤 것을 쓸지 | Amdahl's Law, Gustafson's Law |
| 스레드 생명주기 | New → Runnable → Running → Blocked → Terminated | 스케줄링 알고리즘 |
| 컨텍스트 스위칭 | 오버헤드, 레지스터 저장/복원 | CPU 캐시 영향, TLB flush |

### 2. 동기화 기법 (02-synchronization/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| Mutex / Lock | 상호 배제, 임계 영역 보호 | Spinlock vs Mutex, Recursive Lock |
| Semaphore | 카운팅, 바이너리 세마포어 | Dijkstra's 알고리즘 |
| Condition Variable | 조건 대기, 신호 전달 | Spurious wakeup |
| Atomic Operations | CAS, Fetch-and-Add | Memory Model |
| Memory Barrier | Acquire/Release, Full Fence | CPU 아키텍처별 차이 |
| Reader-Writer Lock | 읽기 우선 vs 쓰기 우선 | 성능 특성 |

### 3. 동시성 문제 (03-concurrency-problems/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| Race Condition | 원인, 탐지, 해결 | TOCTOU (Time-of-check to time-of-use) |
| Deadlock | 4가지 조건, 예방/회피/탐지/복구 | Banker's Algorithm |
| Livelock | Deadlock과의 차이, 해결 방법 | 실제 사례 분석 |
| Starvation | 원인과 해결 (Fairness) | Fair Lock 구현 |
| Priority Inversion | Mars Pathfinder 사례 | Priority Inheritance Protocol |

### 4. 동시성 패턴 (04-concurrency-patterns/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| Producer-Consumer | Bounded Buffer, Backpressure | 다중 생산자/소비자 |
| Reader-Writer | 락 기반 구현 | Lock-free 구현 |
| Thread Pool | 작업 큐, 워커 스레드 | Work Stealing |
| Actor Model | 메시지 패싱, 상태 캡슐화 | Akka, Erlang OTP |
| Future/Promise | 비동기 결과 처리 | Continuation, Composition |
| Pipeline | 스테이지 연결 | 파이프라인 병렬화 |
| Fan-Out/Fan-In | 병렬 분산, 결과 수집 | 로드 밸런싱 |

### 5. Lock-Free 프로그래밍 (05-lock-free-programming/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| CAS | Compare-And-Swap 원리 | LL/SC (Load-Linked/Store-Conditional) |
| Memory Ordering | Sequential Consistency, Relaxed | C++11 Memory Model |
| Lock-Free Queue | Michael-Scott Queue | MPMC Queue |
| Lock-Free Stack | Treiber Stack | Backoff 전략 |
| ABA Problem | 원인과 해결책 | Versioned Pointers |
| Hazard Pointers | Safe Memory Reclamation | Epoch-based Reclamation |

### 6. 언어별 구현 (06-language-implementations/)

#### C++ (std::thread 기반)
- RAII 기반 락 관리 (`lock_guard`, `unique_lock`, `scoped_lock`)
- `std::atomic` 활용법
- `std::async`와 `std::future`
- C++20 Coroutines 소개

#### C# (Task 기반)
- Task vs Thread 비교
- async/await 패턴과 주의사항
- `SemaphoreSlim`으로 비동기 락
- `Concurrent Collections` 활용

#### Go (Goroutine/Channel 기반)
- CSP 모델 철학: "공유 메모리로 통신하지 말고, 통신으로 메모리를 공유하라"
- Channel 패턴 (Buffered/Unbuffered)
- `sync.WaitGroup`, `sync.Mutex`
- `context.Context`로 취소 처리

#### JavaScript (Event Loop 기반)
- Single-threaded 본질과 Event Loop
- Web Workers (브라우저)
- Worker Threads (Node.js)
- `SharedArrayBuffer`와 `Atomics`

### 7. 게임 서버 적용 (07-game-server-applications/)

| 문서 | 내용 |
|------|------|
| 실시간 게임서버 | MMO, MOBA, FPS 서버의 스레딩 모델, 밀리초 단위 응답 요구사항 |
| 비실시간 서버 | 웹 서버, REST API, 데이터베이스 연동시 스레딩 |
| MMO 아키텍처 | 분산 서버, 샤딩, 월드 파티셔닝 |
| Game Loop | 고정 프레임 게임 루프와 멀티스레딩 조합 |
| 네트워킹 | I/O 멀티플렉싱, IOCP, epoll과 스레드 풀 |

### 8. 오픈소스 분석 (08-open-source-analysis/)

| 프로젝트 | 언어 | 분석 포인트 |
|----------|------|-------------|
| [libcds](https://github.com/khizmax/libcds) | C++ | Lock-free 데이터 구조, Hazard Pointers |
| [Folly](https://github.com/facebook/folly) | C++ | MPMC Queue, 고성능 동시성 유틸리티 |
| [Nakama](https://heroiclabs.com/nakama/) | Go | 게임 서버 스케일링, 200만 CCU |
| [Colyseus](https://colyseus.io/) | Node.js | 상태 동기화, 매치메이킹 |
| [xsync](https://github.com/puzpuzpuz/xsync) | Go | 동시성 데이터 구조 |
| [xenium](https://github.com/mpoeter/xenium) | C++ | Memory Reclamation 기법들 |

### 9. 플랫폼 차이 (10-platform-differences/)

| 문서 | 내용 | 심화 주제 |
|------|------|-----------|
| 스레드 생성 | CreateThread vs pthread_create | 스레드 속성, Detached vs Joinable |
| 동기화 프리미티브 | CRITICAL_SECTION vs pthread_mutex_t, Event 구현 | WaitForMultipleObjects 대체 |
| TLS | TlsAlloc vs pthread_key_create | __declspec(thread) vs __thread |
| IPC | Named Objects vs POSIX IPC | Shared Memory, Pipes, Message Queue |
| 스케줄링 | Priority Classes vs Nice Values | Thread Affinity, Real-Time Scheduling |
| 에러 처리 | GetLastError vs errno | 스레드 안전 에러 처리 |
| 이식성 레이어 | C++11 std::thread, 조건부 컴파일 | CMake 플랫폼 감지 |
| 성능 비교 | 벤치마크 결과 | 플랫폼별 최적화 전략 |

---

## 📊 언어별 비교 요약 (미리보기)

| 특성 | C++ | C# | Go | JavaScript |
|------|-----|----|----|------------|
| **스레드 모델** | OS Thread | OS Thread + TPL | Goroutine (M:N) | Event Loop + Workers |
| **기본 동기화** | std::mutex | lock / Monitor | sync.Mutex | Atomics |
| **비동기 패턴** | std::async/future | async/await | Channel | async/await |
| **메모리 관리** | 수동 (스마트 포인터) | GC | GC | GC |
| **Lock-Free 지원** | std::atomic (강력) | Interlocked | atomic 패키지 | Atomics (제한적) |
| **철학** | 저수준 제어 | 생산성 + 성능 균형 | 간결한 동시성 | 비동기 I/O 중심 |

---

## 🎯 학습 순서 권장

```
1. 기초 개념 (필수)
   ↓
2. 동기화 기법 (필수)
   ↓
3. 동시성 문제 (필수)
   ↓
4. 동시성 패턴 (권장)
   ↓
5. 언어별 구현 (필요에 따라 선택)
   ↓
6. Lock-Free 프로그래밍 (고급)
   ↓
7. 게임 서버 적용 (도메인 특화)
   ↓
8. 오픈소스 분석 (심화 학습)
   ↓
9. 플랫폼 차이 (크로스 플랫폼 개발 시 필수)
```

---

## 📚 주요 참고 자료

### 온라인 리소스
- [Educative - Multithreading Fundamentals](https://www.educative.io/blog/multithreading-and-concurrency-fundamentals)
- [Java Concurrency Tutorial (Jenkov)](https://jenkov.com/tutorials/java-concurrency/index.html)
- [awesome-lockfree](https://github.com/rigtorp/awesome-lockfree)
- [Go Concurrency Patterns (Go Blog)](https://go.dev/blog/pipelines)
- [C++ Core Guidelines - Concurrency](https://www.modernescpp.com/index.php/c-core-guidelines-sharing-data-between-threads/)

### 게임 서버 관련
- [Multi-Threaded Game Server Design](https://spirited.io/multi-threaded-game-server-design/)
- [Vulkan Guide - Multithreading](https://vkguide.dev/docs/extra-chapter/multithreading/)
- [Agones - Game Server Scaling](https://agones.dev/site/docs/overview/)

### 서적 (권장)
- "C++ Concurrency in Action" - Anthony Williams
- "Java Concurrency in Practice" - Brian Goetz
- "Concurrency in Go" - Katherine Cox-Buday
- "The Art of Multiprocessor Programming" - Herlihy & Shavit

---

## ⏱️ 작업 체크리스트

- [x] README.md 메인 페이지 작성
- [x] 01-fundamentals/ 섹션 완성
- [x] 02-synchronization/ 섹션 완성
- [x] 03-concurrency-problems/ 섹션 완성
- [x] 04-concurrency-patterns/ 섹션 완성
- [x] 05-lock-free-programming/ 섹션 완성
- [x] 06-language-implementations/ 섹션 완성
- [x] 07-game-server-applications/ 섹션 완성
- [x] 08-open-source-analysis/ 섹션 완성
- [x] 09-appendix/ 섹션 완성
- [x] 10-platform-differences/ 섹션 완성
- [ ] 예시 코드 작성 및 테스트

---

## 🔧 예시 코드 언어

각 예시 코드는 다음 언어들로 제공됩니다:
- **C++17/20** - 가장 상세한 예시
- **C#** - .NET 6+ 기준
- **Go** - 최신 버전
- **JavaScript/TypeScript** - ES2022+, Node.js 18+

---

*이 계획은 조사된 온라인 자료와 베스트 프랙티스를 기반으로 작성되었습니다.*
