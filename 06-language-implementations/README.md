# 프로그래밍 언어별 Concurrency 구현

이 섹션에서는 다양한 프로그래밍 언어가 concurrent 및 parallel 프로그래밍을 어떻게 구현하는지 종합적으로 비교합니다. 이러한 차이점을 이해하면 사용 사례에 적합한 언어를 선택하고 언어 간 지식을 전환하는 데 도움이 됩니다.

## 개요

각 언어는 다음 요소를 기반으로 고유한 concurrency 접근 방식을 발전시켜 왔습니다:
- 언어 철학과 설계 목표
- 역사적 맥락과 발전 과정
- 대상 사용 사례와 도메인
- 성능 요구사항
- 안전성과 사용 편의성 간의 trade-off

## 다루는 언어

- **[C++](./cpp/)** - 세밀한 제어가 가능한 시스템 프로그래밍
- **[C#](./csharp/)** - 높은 수준의 추상화를 갖춘 엔터프라이즈 개발
- **[Go](./go/)** - first-class citizen으로서의 concurrent 프로그래밍
- **[JavaScript](./javascript/)** - Event-driven, single-threaded concurrency

## 종합 언어 비교

### Concurrency Model 비교

| 특성 | C++ | C# | Go | JavaScript |
|---------|-----|----|----|------------|
| **주요 Model** | OS threads with shared memory | Tasks and async/await | Goroutines (green threads) | Event loop with async |
| **Thread 유형** | 1:1 (OS threads) | Hybrid (threadpool + async) | M:N (multiplexed) | Single-threaded (mostly) |
| **Thread 생성** | `std::thread` | `Thread`, `Task` | `go` keyword | N/A (Workers for true parallelism) |
| **Lightweight Threads** | No | No | Yes (goroutines) | No |
| **기본 Stack 크기** | ~2MB per thread | ~1MB per thread | ~2KB per goroutine (growable) | N/A |
| **최대 동시 실행 단위** | Thousands | Thousands | Millions | Thousands (via workers) |

### Synchronization Primitives

| Primitive | C++ | C# | Go | JavaScript |
|-----------|-----|----|----|------------|
| **Mutex** | `std::mutex` | `lock`, `Monitor` | `sync.Mutex` | N/A (single-threaded) |
| **Read-Write Lock** | `std::shared_mutex` | `ReaderWriterLockSlim` | `sync.RWMutex` | N/A |
| **Semaphore** | `std::counting_semaphore` (C++20) | `SemaphoreSlim` | `chan` (buffer) | N/A |
| **Condition Variable** | `std::condition_variable` | `Monitor.Wait/Pulse` | `sync.Cond` | N/A |
| **Atomic Operations** | `std::atomic<T>` | `Interlocked`, `Volatile` | `sync/atomic` | `Atomics` (SharedArrayBuffer) |
| **Barriers** | `std::barrier` (C++20) | `Barrier` | `sync.WaitGroup` | N/A |

### 통신 메커니즘

| 메커니즘 | C++ | C# | Go | JavaScript |
|-----------|-----|----|----|------------|
| **Message Passing** | No built-in | `Channel<T>` (.NET Core) | `chan` (built-in) | `postMessage()` |
| **Shared Memory** | Yes (default) | Yes (default) | Yes (discouraged) | `SharedArrayBuffer` |
| **Channels** | No (third-party) | `System.Threading.Channels` | First-class `chan` | No (messages only) |
| **Memory Model** | C++11 memory model | CLI memory model | Go memory model | Sequential consistency |

### Async 프로그래밍

| 특성 | C++ | C# | Go | JavaScript |
|---------|-----|----|----|------------|
| **Async 문법** | `std::async`, `std::future` | `async`/`await` | Implicit (goroutines) | `async`/`await` |
| **Promise/Future** | `std::future`, `std::promise` | `Task<T>` | N/A (channels) | `Promise` |
| **Cancellation** | `std::stop_token` (C++20) | `CancellationToken` | `context.Context` | `AbortController` |
| **Timeout 지원** | Manual | `Task.Delay`, `CancellationToken` | `context.WithTimeout` | `Promise.race` |
| **Error Handling** | Exceptions in futures | Try-catch with async | Multiple return values | Try-catch with async |

### Data Structures

| 구조 | C++ | C# | Go | JavaScript |
|-----------|-----|----|----|------------|
| **Concurrent Queue** | `std::queue` + mutex | `ConcurrentQueue<T>` | `chan` | N/A |
| **Concurrent Map** | `std::map` + mutex | `ConcurrentDictionary<K,V>` | `sync.Map` | N/A |
| **Concurrent Set** | `std::set` + mutex | `ConcurrentBag<T>` | map + mutex | N/A |
| **Lock-Free Structures** | Manual with atomics | `Concurrent*` collections | Manual with atomics | Manual with Atomics |

### 성능 특성

| 항목 | C++ | C# | Go | JavaScript |
|--------|-----|----|----|------------|
| **Thread 생성 비용** | High (~100us) | High (~100us) | Very Low (~1us) | N/A |
| **Context Switch 비용** | High (OS scheduler) | High (OS scheduler) | Low (runtime scheduler) | N/A |
| **Memory Overhead** | High (per thread) | High (per thread) | Low (per goroutine) | Low |
| **확장성** | Limited by OS | Limited by OS | Excellent | Limited |
| **Raw Performance** | Excellent | Very Good | Very Good | Good |

### 안전성 및 사용 편의성

| 특성 | C++ | C# | Go | JavaScript |
|---------|-----|----|----|------------|
| **Data Race 감지** | ThreadSanitizer | No built-in | `go run -race` | N/A (mostly) |
| **Deadlock 감지** | No | No | Runtime detection | No |
| **Memory Safety** | No (manual management) | Yes (GC) | Yes (GC) | Yes (GC) |
| **Type Safety** | Strong, static | Strong, static | Strong, static | Weak, dynamic |
| **학습 난이도** | Steep | Moderate | Gentle | Gentle |
| **추상화 수준** | Low to High | High | Medium | High |

### Ecosystem 및 Tooling

| 도구/기능 | C++ | C# | Go | JavaScript |
|--------------|-----|----|----|------------|
| **Standard Library** | Comprehensive (C++11+) | Very comprehensive | Minimalist but complete | Comprehensive |
| **Debugging 도구** | gdb, lldb, VS debugger | Visual Studio, VS Code | Delve, VS Code | Chrome DevTools, VS Code |
| **Profiling 도구** | perf, valgrind, gprof | dotTrace, PerfView | pprof (built-in) | Chrome DevTools |
| **Static Analysis** | clang-tidy, cppcheck | Roslyn analyzers | go vet, staticcheck | ESLint |
| **Package Manager** | vcpkg, conan | NuGet | go modules (built-in) | npm, yarn |

## 언어 철학 비교

### C++: 최대한의 제어와 성능

**철학**: "사용하지 않는 것에 대해 비용을 지불하지 않는다"

**강점**:
- Zero-cost abstractions
- Memory와 thread에 대한 세밀한 제어
- 시스템 프로그래밍과 고성능 컴퓨팅에 탁월
- 강력한 template metaprogramming

**약점**:
- 복잡하고 장황함
- 미묘한 버그를 도입하기 쉬움
- 수동 memory 관리 필요
- 가파른 학습 곡선

**적합한 분야**: 게임 엔진, 운영 체제, 임베디드 시스템, 고빈도 트레이딩

### C#: 생산성과 엔터프라이즈 기능

**철학**: "일반적인 경우는 쉽게, 어려운 경우도 가능하게"

**강점**:
- 우수한 async/await 문법
- Concurrent collections을 포함한 풍부한 standard library
- 강력한 tooling 및 IDE 지원
- 안전성과 성능의 균형이 좋음

**약점**:
- Windows 중심적 (.NET Core로 개선 중)
- GC pause가 문제가 될 수 있음
- C++보다 제어 수준이 낮음
- 플랫폼 제한

**적합한 분야**: 엔터프라이즈 애플리케이션, 웹 서비스, 데스크탑 애플리케이션, 게임 개발 (Unity)

### Go: 단순함과 내장 Concurrency

**철학**: "Concurrency는 parallelism이 아니지만, parallelism을 가능하게 한다"

**강점**:
- Goroutine이 concurrent 프로그래밍을 자연스럽게 만듦
- Channel이 안전한 통신을 제공
- 단순한 언어 설계
- 확장 가능한 네트워크 서비스에 탁월
- 빠른 컴파일과 배포

**약점**:
- 다른 언어에 비해 표현력이 부족
- Generics 미지원 (Go 1.18까지)
- 스케줄링에 대한 제한적 제어
- GC로 인한 지연 스파이크 가능

**적합한 분야**: Microservices, 네트워크 서버, 분산 시스템, 클라우드 인프라

### JavaScript: Event-Driven 단순함

**철학**: "메인 thread를 절대 차단하지 않는다"

**강점**:
- I/O-bound 작업에 자연스러움
- 단순한 mental model (event loop)
- 우수한 async/await 문법
- 어디서나 실행 가능 (유비쿼터스)

**약점**:
- Single-threaded 메인 실행
- Worker가 무겁고 사용하기 불편함
- CPU-intensive 작업에 부적합
- 진정한 shared memory 없음 (대부분)

**적합한 분야**: 웹 애플리케이션, Node.js 서버, I/O-bound 서비스, UI 프로그래밍

## 각 언어를 선택해야 하는 경우

### C++를 선택해야 하는 경우:
- 최대 성능과 제어가 필요할 때
- 시스템 소프트웨어나 게임 엔진을 구축할 때
- 엄격한 latency 요구사항이 있을 때
- 하드웨어나 OS API와 인터페이스해야 할 때
- Memory 레이아웃과 접근 패턴이 중요할 때

### C#를 선택해야 하는 경우:
- 엔터프라이즈 애플리케이션을 구축할 때
- 좋은 성능과 함께 빠른 개발을 원할 때
- 우수한 tooling과 IDE 지원이 필요할 때
- Microsoft ecosystem에 있을 때
- 생산성과 성능의 균형을 원할 때

### Go를 선택해야 하는 경우:
- 네트워크 서비스나 microservices를 구축할 때
- 많은 concurrent 연결을 처리해야 할 때
- 단순한 배포를 원할 때 (single binary)
- 가독성과 유지보수성을 우선시할 때
- 클라우드 네이티브 애플리케이션을 구축할 때

### JavaScript를 선택해야 하는 경우:
- 웹 애플리케이션을 구축할 때 (프론트엔드 또는 백엔드)
- I/O-bound 작업을 할 때 (웹 서버, API)
- 클라이언트와 서버 간 코드를 공유해야 할 때
- UI 애플리케이션을 구축할 때
- 워크로드가 본질적으로 event-driven일 때

## 언어 간 공통 패턴

### Producer-Consumer

각 언어는 이 기본 패턴을 다르게 구현합니다:
- **C++**: Mutex와 condition variable로 보호된 queue
- **C#**: `BlockingCollection<T>` 또는 channels
- **Go**: Buffered 또는 unbuffered channels
- **JavaScript**: Event emitters 또는 async iterators

### Worker Pool

- **C++**: Work queue를 가진 thread pool
- **C#**: `Task.Run()`과 TPL 또는 custom thread pool
- **Go**: 공유 channel에서 읽는 여러 goroutines
- **JavaScript**: Message passing을 사용하는 worker threads

### Pipeline

- **C++**: Worker threads를 가진 queue 체인
- **C#**: `System.Threading.Channels` 또는 TPL Dataflow
- **Go**: Goroutine을 가진 channel 체인
- **JavaScript**: Transform streams 또는 async generators

## 학습 경로 추천

### C++을 알고 있다면:
1. **Go를 다음으로 시도**: 더 간단한 문법, 내장 concurrency, 유사한 성능 도메인
2. **그 다음 C#**: 더 높은 수준의 추상화, 더 나은 async/await
3. **마지막으로 JavaScript**: 다른 패러다임, event-driven model

### C#을 알고 있다면:
1. **JavaScript를 다음으로 시도**: 유사한 async/await 문법, 다른 runtime model
2. **그 다음 Go**: 단순하지만 강력한 concurrency primitives
3. **마지막으로 C++**: 저수준 세부사항에 대한 깊은 이해

### Go를 알고 있다면:
1. **JavaScript를 다음으로 시도**: 다른 concurrency model, event loop
2. **그 다음 C#**: 더 많은 기능, 유사한 철학
3. **마지막으로 C++**: 최대한의 제어와 성능

### JavaScript를 알고 있다면:
1. **C#를 다음으로 시도**: 유사한 async/await, strong typing 추가
2. **그 다음 Go**: 단순하고 강력한 concurrency
3. **마지막으로 C++**: 완전한 시스템 프로그래밍 역량

## 핵심 요약

1. **만능 해결책은 없다**: 각 언어는 서로 다른 도메인에서 뛰어남
2. **Trade-off가 중요하다**: 성능 vs. 안전성 vs. 생산성
3. **Concurrency ≠ Parallelism**: 각 언어에서의 차이를 이해하라
4. **높은 수준부터 시작하라**: 저수준 primitive에 뛰어들기 전에 async/await 패턴부터 시작
5. **추측하지 말고 측정하라**: 구체적인 사용 사례에서 profile 및 benchmark를 수행

## 추가 자료

### 여러 언어에 걸친 자료
- "Seven Concurrency Models in Seven Weeks" by Paul Butcher
- "The Art of Multiprocessor Programming" by Herlihy and Shavit
- "Programming Language Pragmatics" by Scott

### 언어별 심화 학습
- [C++ Concurrency](./cpp/)
- [C# Concurrency](./csharp/)
- [Go Concurrency](./go/)
- [JavaScript Concurrency](./javascript/)

## 기여하기

새로운 예제나 언어를 추가할 때:
1. 기존 구조를 따르세요 (README + topic 파일)
2. 작동하는 코드 예제를 포함하세요
3. 다른 언어와의 비교를 추가하세요
4. 흔한 함정을 문서화하세요
5. 비교 테이블을 업데이트하세요

---

**다음 단계**: 개별 언어 구현으로 들어가 실용적인 예제와 best practices를 살펴보세요.
