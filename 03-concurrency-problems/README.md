# 동시성 문제

## 개요

동시성 프로그래밍은 순차적 프로그래밍에서는 존재하지 않는 부류의 문제를 야기합니다. 이러한 문제들은 thread 실행의 비결정적 인터리빙과 공유 자원 접근에서 발생합니다. 올바르고 효율적인 멀티스레드 프로그램을 작성하려면 이러한 문제들을 이해하는 것이 매우 중요합니다.

## 다섯 가지 주요 동시성 문제

### 1. Race Condition
여러 thread가 공유 데이터에 동시에 접근하고, 그 중 하나 이상이 데이터를 수정하면서 적절한 동기화가 이루어지지 않을 때 발생합니다.

**주요 특징:**
- 비결정적 동작
- 결과가 타이밍에 의존
- 재현이 어려움

**자세히 알아보기:** [01-race-condition.md](./01-race-condition.md)

### 2. Deadlock
두 개 이상의 thread가 서로가 보유한 자원을 기다리면서 영구적으로 차단되는 상태입니다.

**주요 특징:**
- 완전한 정지 상태
- 순환 의존성
- 해결하려면 외부 개입이 필요

**자세히 알아보기:** [02-deadlock.md](./02-deadlock.md)

### 3. Livelock
thread들이 서로에 대한 응답으로 지속적으로 상태를 변경하지만 실질적인 진행이 이루어지지 않는 상태입니다.

**주요 특징:**
- thread가 활성 상태이지만 비생산적
- 전진이 없음
- 자원을 많이 소비

**자세히 알아보기:** [03-livelock.md](./03-livelock.md)

### 4. Starvation
thread가 필요한 자원에 대한 접근을 영구적으로 거부당하는 상태입니다.

**주요 특징:**
- 불공정한 스케줄링
- 일부 thread는 진행되지만 다른 thread는 진행되지 않음
- 성능 저하를 유발할 수 있음

**자세히 알아보기:** [04-starvation.md](./04-starvation.md)

### 5. Priority Inversion
높은 우선순위의 thread가 낮은 우선순위의 thread가 자원을 해제하기를 기다리며 차단되는 상태입니다.

**주요 특징:**
- 우선순위 의미를 위반
- 치명적인 장애를 유발할 수 있음
- 해결하려면 priority inheritance가 필요

**자세히 알아보기:** [05-priority-inversion.md](./05-priority-inversion.md)

## 문제 비교 매트릭스

```
┌─────────────────────┬──────────────┬────────────┬─────────────┬──────────────┐
│      Problem        │   Progress   │  Resource  │   Detection │  Severity    │
│                     │              │   Usage    │  Difficulty │              │
├─────────────────────┼──────────────┼────────────┼─────────────┼──────────────┤
│ Race Condition      │ Unpredictable│    Normal  │     High    │   Critical   │
│ Deadlock            │     None     │    Locked  │    Medium   │   Critical   │
│ Livelock            │     None     │     High   │    Medium   │     High     │
│ Starvation          │   Partial    │   Unfair   │      Low    │    Medium    │
│ Priority Inversion  │   Delayed    │    Locked  │    Medium   │     High     │
└─────────────────────┴──────────────┴────────────┴─────────────┴──────────────┘
```

## 일반적인 예방 전략

### 1. 공유 상태 최소화
- 공유 메모리보다 메시지 전달을 선호
- thread-local storage 사용
- 불변성을 고려한 설계

### 2. 적절한 동기화 사용
- Lock, mutex, semaphore
- Atomic 연산
- Memory barrier

### 3. 모범 사례 따르기
- Lock 순서 규약
- Timeout 메커니즘
- 공정한 스케줄링 정책

### 4. 테스트 및 검증
- 스트레스 테스트
- Race 탐지 도구 (ThreadSanitizer, Helgrind)
- 형식 검증 방법

## 문제를 야기하는 일반적인 패턴

### 패턴 1: Check-Then-Act
```c
// WRONG: Race condition
if (resource->available) {
    // Another thread might grab it here!
    resource->use();
}

// CORRECT: Atomic check-and-act
lock(mutex);
if (resource->available) {
    resource->use();
}
unlock(mutex);
```

### 패턴 2: 다중 Lock 획득
```c
// WRONG: Potential deadlock
lock(mutex_a);
lock(mutex_b);  // Another thread might lock in reverse order

// CORRECT: Consistent lock ordering
lock(min(mutex_a, mutex_b));
lock(max(mutex_a, mutex_b));
```

### 패턴 3: Lock + Wait
```c
// WRONG: Can cause deadlock
lock(mutex);
wait_for_event();  // Holding lock while waiting

// CORRECT: Release lock before waiting
lock(mutex);
unlock(mutex);
wait_for_event();
```

## 탐지 도구

### 정적 분석
- **Clang Thread Safety Analysis**: 컴파일 시점 탐지
- **Coverity**: 상용 정적 분석기
- **Infer**: Facebook의 정적 분석기

### 동적 분석
- **ThreadSanitizer (TSan)**: Race condition 탐지기
- **Helgrind**: Valgrind의 thread 오류 탐지기
- **DRD**: Data race 탐지기

### 프로파일링 도구
- **perf**: Linux 성능 프로파일러
- **VTune**: Intel의 성능 프로파일러
- **gprof**: GNU 프로파일러

## 실제 사례의 영향

### 유명한 사고들

1. **Therac-25 (1985-1987)**
   - 방사선 치료 기기의 race condition
   - 결과: 치명적인 방사선 과다 노출
   - 원인: 공유 상태에 대한 동시 접근

2. **미국 북동부 대정전 (2003)**
   - 경보 시스템의 race condition
   - 결과: 5천만 명이 전력 공급을 받지 못함
   - 원인: 보호되지 않은 공유 데이터 구조

3. **Mars Pathfinder (1997)**
   - Priority inversion 문제
   - 결과: 화성에서 시스템 리셋 발생
   - 원인: 낮은 우선순위의 thread가 높은 우선순위의 thread를 차단

4. **Knight Capital (2012)**
   - 거래 소프트웨어의 race condition
   - 결과: 45분 만에 4억 4천만 달러 손실
   - 원인: 주문 상태의 동시 수정

## 테스트 전략

### 1. 스트레스 테스트
```bash
# Run with maximum threads
./program --threads=1000 --iterations=10000000

# Use CPU stress
stress --cpu 8 --timeout 60s & ./program
```

### 2. Race 탐지
```bash
# Compile with ThreadSanitizer
gcc -fsanitize=thread -g program.c -o program

# Run with Helgrind
valgrind --tool=helgrind ./program
```

### 3. 결정적 테스트
```c
// Use barriers to force specific interleavings
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 2);

// Thread 1
critical_section_1();
pthread_barrier_wait(&barrier);  // Sync point

// Thread 2
pthread_barrier_wait(&barrier);  // Sync point
critical_section_2();
```

## 학습 경로

1. **여기서 시작**: Race condition을 철저히 이해하기
2. **기초 다지기**: Deadlock과 예방법 학습
3. **심화 주제**: Livelock과 starvation 연구
4. **실시간 시스템**: Priority inversion 마스터하기

## 연습 문제

각 섹션에는 실습 연습 문제가 포함되어 있습니다. 순서대로 진행하세요:

1. **Race Condition**: thread-safe 카운터 구현
2. **Deadlock**: 식사하는 철학자 문제 해결
3. **Livelock**: 복도 문제 해결
4. **Starvation**: 공정한 reader-writer lock 구현
5. **Priority Inversion**: Priority inheritance 시뮬레이션

## 추가 자료

### 도서
- "The Art of Multiprocessor Programming" by Herlihy & Shavit
- "Java Concurrency in Practice" by Goetz et al.
- "Programming with POSIX Threads" by Butenhof

### 논문
- "Dining Philosophers Problem" - Dijkstra (1965)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)

### 온라인 자료
- POSIX Threads Programming (LLNL Tutorial)
- MIT 6.826: Principles of Computer Systems
- CMU 15-410: Operating System Design

## 다음 단계

이 섹션을 완료한 후에는 다음을 할 수 있어야 합니다:
1. 다섯 가지 주요 동시성 문제를 모두 이해
2. 코드에서 이러한 문제를 식별할 수 있음
3. 예방 및 탐지 전략을 숙지
4. 이러한 문제를 해결하는 실무 경험 보유

계속 진행: **04-synchronization-primitives/** 에서 이러한 문제를 해결하기 위한 도구를 학습합니다.
