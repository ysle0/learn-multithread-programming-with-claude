# Concurrency Patterns

## 개요

Concurrency pattern은 concurrent 및 parallel 시스템을 설계할 때 흔히 마주치는 문제들에 대한 검증된 해결책입니다. 이러한 pattern은 조율, 통신, 자원 관리에 대해 충분히 검증된 접근 방식을 제공하여, 개발자가 더 안전하고 효율적이며 유지보수하기 쉬운 multithread 코드를 작성할 수 있도록 도와줍니다.

## Concurrency Pattern이 중요한 이유

- **재사용성**: 다양한 애플리케이션에서 활용 가능한 검증된 해결책
- **의사소통**: concurrent 설계를 논의하기 위한 공통 용어
- **효율성**: 일반적인 concurrency 문제에 대한 최적화된 접근 방식
- **안전성**: race condition, deadlock 및 기타 concurrency 버그를 방지하는 데 도움이 되는 pattern
- **확장성**: 병렬 처리가 증가해도 효과적으로 확장되는 설계

## Pattern 분류

### 1. 통신 Pattern
thread 간 통신과 데이터 교환 방식에 중점을 둔 pattern:
- **Producer-Consumer**: 데이터 생산자와 소비자의 분리
- **Pipeline**: 여러 순차적 단계를 통한 데이터 처리
- **Actor Model**: 독립적인 actor 간의 message passing

### 2. 자원 관리 Pattern
공유 자원을 관리하기 위한 pattern:
- **Reader-Writer**: concurrent read/write 접근 최적화
- **Thread Pool**: thread를 재사용하여 효율적으로 작업 실행

### 3. 조율 Pattern
thread 간 작업을 조율하기 위한 pattern:
- **Future/Promise**: 비동기 연산의 최종 결과를 표현
- **Fan-Out/Fan-In**: 작업 분배 및 결과 수집

## 다루는 Pattern

### 01. Producer-Consumer Pattern
```
[Producers] --> [Shared Queue] --> [Consumers]
```
**사용 시점**: 데이터 생산과 소비를 분리해야 하거나, 생산/소비 속도가 다를 때, 또는 작업 큐를 구현해야 할 때 사용합니다.

**핵심 개념**: bounded buffer, blocking 연산, backpressure

### 02. Reader-Writer Pattern
```
Multiple Readers (concurrent)
     OR
Single Writer (exclusive)
```
**사용 시점**: read 연산이 write보다 훨씬 많거나, write 안전성을 보장하면서 read 처리량을 최적화해야 할 때 사용합니다.

**핵심 개념**: shared/exclusive lock, read 편향 vs write 편향, 공정성

### 03. Thread Pool Pattern
```
[Tasks] --> [Task Queue] --> [Worker Threads]
```
**사용 시점**: thread 생성 비용이 클 때, 짧은 수명의 작업이 많을 때, 또는 동시 실행을 제한해야 할 때 사용합니다.

**핵심 개념**: work stealing, task scheduling, thread 수명 주기 관리

### 04. Actor Model
```
[Actor A] <--messages--> [Actor B] <--messages--> [Actor C]
```
**사용 시점**: 공유 상태를 제거하고자 할 때, 위치 투명성이 필요할 때, 또는 분산 시스템을 구축할 때 사용합니다.

**핵심 개념**: message passing, actor 격리, mailbox

### 05. Future/Promise Pattern
```
Promise (Writer) --> Shared State <-- Future (Reader)
```
**사용 시점**: 비동기적으로 계산되는 값을 표현해야 하거나, 연산을 연결하거나, 비동기 오류를 처리해야 할 때 사용합니다.

**핵심 개념**: 지연 연산, continuation passing, async/await

### 06. Pipeline Pattern
```
[Stage 1] --> [Stage 2] --> [Stage 3] --> [Output]
```
**사용 시점**: 처리를 순차적 단계로 나눌 수 있을 때, 각 단계가 동시에 실행될 수 있을 때, 또는 stream 처리가 필요할 때 사용합니다.

**핵심 개념**: stage 병렬 처리, stage 간 buffering, backpressure

### 07. Fan-Out/Fan-In Pattern
```
           --> [Worker 1] --
[Input] --> --> [Worker 2] --> --> [Aggregator]
           --> [Worker 3] --
```
**사용 시점**: 작업을 독립적인 하위 작업으로 분할할 수 있을 때, 병렬 연산의 결과를 집계해야 할 때 사용합니다.

**핵심 개념**: 작업 분배, 결과 집계, synchronization

## Pattern 선택 가이드

### Producer-Consumer를 선택해야 할 때:
- 데이터 생산 속도와 소비 속도가 다를 때
- 컴포넌트 간 buffering이 필요할 때
- producer와 consumer를 분리하면 설계가 개선될 때

### Reader-Writer를 선택해야 할 때:
- read가 write보다 압도적으로 많을 때 (90% 이상이 read)
- read 연산이 안전하게 동시에 수행될 수 있을 때
- write 연산에 exclusive 접근이 필요할 때

### Thread Pool을 선택해야 할 때:
- 짧은 시간 동안 수행되는 작업이 많을 때
- thread 생성 오버헤드가 클 때
- 자원 사용량을 제한해야 할 때

### Actor Model을 선택해야 할 때:
- message passing 기반으로 설계할 수 있을 때
- 각 컴포넌트를 격리할 수 있을 때
- 분산 처리/위치 투명성이 필요할 수 있을 때

### Future/Promise를 선택해야 할 때:
- 연산이 비동기적으로 완료될 때
- 비동기 연산을 조합해야 할 때
- 오류 처리가 비동기 호출을 통해 전파되어야 할 때

### Pipeline을 선택해야 할 때:
- 처리가 별개의 단계로 구성될 때
- 각 단계가 서로 다른 항목을 동시에 처리할 수 있을 때
- 데이터가 한 방향으로 흐를 때

### Fan-Out/Fan-In을 선택해야 할 때:
- 작업을 독립적인 청크로 분할할 수 있을 때
- 결과를 집계해야 할 때
- 데이터 병렬 처리를 활용하고자 할 때

## Pattern 조합

더 복잡한 시나리오를 위해 pattern을 조합할 수 있습니다:

- **Thread Pool + Producer-Consumer**: thread pool의 worker가 큐에서 작업을 소비
- **Pipeline + Fan-Out/Fan-In**: 개별 pipeline 단계에서 병렬 처리를 위해 fan-out 수행
- **Actor Model + Future/Promise**: actor가 비동기 요청-응답을 위해 future를 반환
- **Thread Pool + Future/Promise**: thread pool이 promise 값을 설정하는 작업을 실행

## 피해야 할 일반적인 Anti-Pattern

### 1. 과도한 동기화 (Over-Synchronization)
필요 이상의 synchronization을 사용하여 순차 실행이 되어버리는 현상.

### 2. 불충분한 동기화 (Under-Synchronization)
synchronization이 불충분하여 race condition이 발생하는 현상.

### 3. Lock Convoy
thread가 lock에 대기열을 형성하여 병렬 처리가 감소하는 현상.

### 4. Priority Inversion
높은 우선순위의 thread가 낮은 우선순위의 thread를 기다리는 현상.

### 5. Thread Exhaustion
thread를 무제한으로 생성하는 현상.

## 성능 고려사항

### 세분화 수준 (Granularity)
- **너무 거친 경우**: 제한된 병렬 처리, 코어 활용 부족
- **너무 미세한 경우**: synchronization 오버헤드가 유용한 작업을 압도
- **적절한 경우**: 병렬 처리와 오버헤드 간의 균형

### 경합 (Contention)
- **높은 경합**: 많은 thread가 동일한 자원을 두고 경쟁
- **해결 방법**: critical section 크기 줄이기, lock-free 구조 사용, 데이터 분할

### 캐시 효과 (Cache Effects)
- **False Sharing**: 서로 다른 thread가 같은 cache line에 있는 서로 다른 데이터에 접근
- **해결 방법**: padding, alignment, 데이터 구조 설계

## Concurrent Pattern 테스트

### 접근 방법:
1. **Stress Testing**: 높은 thread 수와 부하로 실행
2. **Race Detection**: ThreadSanitizer와 같은 도구 활용
3. **정형 검증 (Formal Verification)**: critical section에 대한 모델 검사
4. **성능 테스트**: 코어 수 증가에 따른 확장성 측정

### 일반적인 문제:
- Deadlock
- Livelock
- Race condition
- Memory ordering 버그
- 자원 누수

## 추가 학습 자료

### 도서
- "Java Concurrency in Practice" by Brian Goetz (원리는 C++에도 적용 가능)
- "Concurrency in C++" by Anthony Williams
- "The Art of Multiprocessor Programming" by Herlihy & Shavit

### 논문
- "Communicating Sequential Processes" by C.A.R. Hoare
- "Actors: A Model of Concurrent Computation" by Gul Agha

### 표준
- C++11/14/17/20 concurrency 기능
- ISO/IEC 14882 (C++ 표준)

## Pattern 구현 참고사항

이 섹션의 각 pattern에는 다음이 포함되어 있습니다:

1. **개념 개요**: 어떤 문제를 해결하는가?
2. **아키텍처 다이어그램**: ASCII art를 사용한 시각적 표현
3. **C++ 구현**: 완전히 동작하는 예제
4. **변형**: 일반적인 변형 및 대안
5. **성능 분석**: 사용 시점, 확장성 특성
6. **일반적인 함정**: 주의해야 할 사항
7. **실제 사례**: 해당 pattern이 실제로 사용되는 곳

## 다음 단계

1. 가장 기본적인 pattern인 **Producer-Consumer**부터 시작하세요
2. 자원 관리를 위한 **Reader-Writer**와 **Thread Pool**로 진행하세요
3. 다른 concurrency 패러다임인 **Actor Model**을 탐구하세요
4. 비동기 프로그래밍을 위한 **Future/Promise**를 학습하세요
5. 복잡한 데이터 처리를 위한 **Pipeline**과 **Fan-Out/Fan-In**을 학습하세요

각 pattern은 이전 섹션(synchronization primitive, memory model, lock-free 프로그래밍)의 개념을 기반으로 하며, 이러한 저수준 도구를 고수준 설계로 결합하는 방법을 보여줍니다.
