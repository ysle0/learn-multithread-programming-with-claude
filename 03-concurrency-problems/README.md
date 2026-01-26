# 동시성 문제 (Concurrency Problems)

## 📌 개요

동시성 프로그래밍은 순차 프로그래밍에서는 존재하지 않는 독특한 문제들을 발생시킵니다. 이러한 문제들은 스레드 실행의 비결정적 인터리빙(interleaving)과 공유 자원 접근에서 발생합니다. 올바르고 효율적인 멀티스레드 프로그램을 작성하려면 이러한 문제들을 이해하는 것이 필수적입니다.

## 5대 주요 동시성 문제

### 1. Race Condition (경쟁 조건)
여러 스레드가 공유 데이터에 동시에 접근하고, 최소 하나 이상이 데이터를 수정할 때, 적절한 동기화 없이 발생하는 문제입니다.

**핵심 특징:**
- 비결정적 동작
- 결과가 타이밍에 의존
- 재현하기 어려움

**자세히 보기:** [01-race-condition.md](./01-race-condition.md)

### 2. Deadlock (교착 상태)
둘 이상의 스레드가 서로가 보유한 자원을 기다리며 영구적으로 차단되는 상태입니다.

**핵심 특징:**
- 완전한 정지 상태
- 순환 의존성
- 해결하려면 개입 필요

**자세히 보기:** [02-deadlock.md](./02-deadlock.md)

### 3. Livelock (라이브락)
스레드들이 서로의 상태 변화에 계속 반응하지만 진행은 하지 못하는 상태입니다.

**핵심 특징:**
- 스레드는 활성 상태지만 비생산적
- 진전 없음
- 리소스 집약적

**자세히 보기:** [03-livelock.md](./03-livelock.md)

### 4. Starvation (기아 상태)
특정 스레드가 필요한 자원에 영구적으로 접근하지 못하는 상태입니다.

**핵심 특징:**
- 불공정한 스케줄링
- 일부 스레드만 진행, 다른 스레드는 진행 못함
- 성능 저하 유발

**자세히 보기:** [04-starvation.md](./04-starvation.md)

### 5. Priority Inversion (우선순위 역전)
높은 우선순위 스레드가 낮은 우선순위 스레드가 자원을 해제할 때까지 차단되는 상태입니다.

**핵심 특징:**
- 우선순위 의미론 위반
- 치명적인 실패 유발 가능
- Priority Inheritance로 해결 필요

**자세히 보기:** [05-priority-inversion.md](./05-priority-inversion.md)

## 문제 비교 매트릭스

```
┌─────────────────────┬──────────────┬────────────┬─────────────┬──────────────┐
│      문제           │   진행 상태  │  리소스    │   탐지      │  심각도      │
│                     │              │   사용     │  난이도     │              │
├─────────────────────┼──────────────┼────────────┼─────────────┼──────────────┤
│ Race Condition      │ 예측 불가    │    정상    │     높음    │   치명적     │
│ Deadlock            │     없음     │    잠김    │    중간     │   치명적     │
│ Livelock            │     없음     │    높음    │    중간     │     높음     │
│ Starvation          │   부분적     │   불공정   │    낮음     │    중간      │
│ Priority Inversion  │   지연됨     │    잠김    │    중간     │     높음     │
└─────────────────────┴──────────────┴────────────┴─────────────┴──────────────┘
```

## 일반적인 예방 전략

### 1. 공유 상태 최소화
- 공유 메모리보다 메시지 전달 선호
- Thread-Local Storage 사용
- 불변성(Immutability) 설계

### 2. 적절한 동기화 사용
- Lock, Mutex, Semaphore
- Atomic 연산
- Memory Barrier

### 3. 모범 사례 준수
- 락 순서 규약
- 타임아웃 메커니즘
- 공정한 스케줄링 정책

### 4. 테스팅 및 검증
- 스트레스 테스팅
- Race Detection 도구 (ThreadSanitizer, Helgrind)
- 형식 검증 방법

## 문제를 유발하는 일반적인 패턴

### 패턴 1: Check-Then-Act (확인-후-행동)
```c
// ❌ 잘못된 예: Race Condition
if (resource->available) {
    // 여기서 다른 스레드가 가져갈 수 있음!
    resource->use();
}

// ✅ 올바른 예: Atomic Check-and-Act
lock(mutex);
if (resource->available) {
    resource->use();
}
unlock(mutex);
```

### 패턴 2: Multiple Lock Acquisition (다중 락 획득)
```c
// ❌ 잘못된 예: Deadlock 가능성
lock(mutex_a);
lock(mutex_b);  // 다른 스레드가 반대 순서로 잠글 수 있음

// ✅ 올바른 예: 일관된 락 순서
lock(min(mutex_a, mutex_b));
lock(max(mutex_a, mutex_b));
```

### 패턴 3: Lock + Wait (락 + 대기)
```c
// ❌ 잘못된 예: Deadlock 유발 가능
lock(mutex);
wait_for_event();  // 락을 보유한 채 대기

// ✅ 올바른 예: 대기 전 락 해제
lock(mutex);
unlock(mutex);
wait_for_event();
```

## 탐지 도구

### 정적 분석 (Static Analysis)
- **Clang Thread Safety Analysis**: 컴파일 타임 탐지
- **Coverity**: 상용 정적 분석기
- **Infer**: Facebook의 정적 분석기

### 동적 분석 (Dynamic Analysis)
- **ThreadSanitizer (TSan)**: Race Condition 탐지기
- **Helgrind**: Valgrind의 스레드 오류 탐지기
- **DRD**: Data Race 탐지기

### 프로파일링 도구
- **perf**: Linux 성능 프로파일러
- **VTune**: Intel의 성능 프로파일러
- **gprof**: GNU 프로파일러

## 실제 사례

### 유명한 사고들

1. **Therac-25 (1985-1987)**
   - 방사선 치료 기기의 Race Condition
   - 결과: 치명적인 방사선 과다 조사
   - 원인: 공유 상태에 대한 동시 접근

2. **미국 동북부 대정전 (2003)**
   - 알람 시스템의 Race Condition
   - 결과: 5천만 명 정전
   - 원인: 보호되지 않은 공유 데이터 구조

3. **Mars Pathfinder (1997)**
   - Priority Inversion 문제
   - 결과: 화성에서 시스템 리셋
   - 원인: 낮은 우선순위 스레드가 높은 우선순위 스레드 차단

4. **Knight Capital (2012)**
   - 거래 소프트웨어의 Race Condition
   - 결과: 45분 만에 4억 4천만 달러 손실
   - 원인: 주문 상태의 동시 수정

## 테스팅 전략

### 1. 스트레스 테스팅
```bash
# 최대 스레드로 실행
./program --threads=1000 --iterations=10000000

# CPU 스트레스 사용
stress --cpu 8 --timeout 60s & ./program
```

### 2. Race Detection
```bash
# ThreadSanitizer로 컴파일
gcc -fsanitize=thread -g program.c -o program

# Helgrind로 실행
valgrind --tool=helgrind ./program
```

### 3. 결정적 테스팅 (Deterministic Testing)
```c
// Barrier를 사용하여 특정 인터리빙 강제
pthread_barrier_t barrier;
pthread_barrier_init(&barrier, NULL, 2);

// Thread 1
critical_section_1();
pthread_barrier_wait(&barrier);  // 동기화 지점

// Thread 2
pthread_barrier_wait(&barrier);  // 동기화 지점
critical_section_2();
```

## 학습 경로

1. **시작**: Race Condition을 철저히 이해
2. **기초 구축**: Deadlock과 예방법 학습
3. **고급 주제**: Livelock과 Starvation 연구
4. **실시간 시스템**: Priority Inversion 마스터

## 연습 문제

각 섹션에는 실전 연습 문제가 포함되어 있습니다. 순서대로 진행하세요:

1. **Race Condition**: 스레드 안전 카운터 구현
2. **Deadlock**: 식사하는 철학자 문제 해결
3. **Livelock**: 복도 문제 해결
4. **Starvation**: 공정한 Reader-Writer Lock 구현
5. **Priority Inversion**: Priority Inheritance 시뮬레이션

## 추가 리소스

### 도서
- "The Art of Multiprocessor Programming" - Herlihy & Shavit
- "Java Concurrency in Practice" - Goetz 외
- "Programming with POSIX Threads" - Butenhof

### 논문
- "Dining Philosophers Problem" - Dijkstra (1965)
- "Monitors: An Operating System Structuring Concept" - Hoare (1974)
- "Priority Inheritance Protocols" - Sha, Rajkumar, Lehoczky (1990)

### 온라인 리소스
- POSIX Threads Programming (LLNL Tutorial)
- MIT 6.826: Principles of Computer Systems
- CMU 15-410: Operating System Design

## 다음 단계

이 섹션을 완료한 후에는 다음을 할 수 있어야 합니다:
1. 5가지 주요 동시성 문제 모두 이해
2. 코드에서 이러한 문제 식별 가능
3. 예방 및 탐지 전략 숙지
4. 이러한 문제 해결에 대한 실전 경험 보유

계속하기: **04-concurrency-patterns/**에서 이러한 문제를 해결하는 도구를 학습하세요.
