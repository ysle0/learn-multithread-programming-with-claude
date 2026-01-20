# 동시성 vs 병렬성

## 📌 핵심 개념

**동시성(Concurrency)**과 **병렬성(Parallelism)**은 자주 혼동되지만 근본적으로 다른 개념입니다.

> **Rob Pike (Go 언어 창시자)의 정의**:
> - **Concurrency is about dealing with lots of things at once.**
>   (동시성은 많은 일을 한 번에 다루는 것)
> - **Parallelism is about doing lots of things at once.**
>   (병렬성은 많은 일을 동시에 실행하는 것)

---

## 🔍 핵심 차이

### 동시성 (Concurrency)

**정의**: 여러 작업을 **논리적으로 동시에 진행**하는 것처럼 보이게 하는 것

**특징**:
- 단일 코어에서도 가능
- 작업을 작은 단위로 쪼개어 번갈아가며 실행
- **구조(Structure)**에 관한 것

**비유**: 혼자서 여러 요리를 번갈아가며 조리
```
요리사 1명:
[국 끓이기] → [밥 짓기] → [국 끓이기] → [반찬 만들기] → [밥 짓기]
→ 논리적으로는 3가지 요리를 "동시에" 진행
```

### 병렬성 (Parallelism)

**정의**: 여러 작업을 **물리적으로 동시에 실행**하는 것

**특징**:
- 멀티 코어 필수
- 작업을 여러 코어에서 동시에 수행
- **실행(Execution)**에 관한 것

**비유**: 여러 요리사가 각자 요리
```
요리사 3명:
요리사 1: [국 끓이기 ─────────────]
요리사 2:          [밥 짓기 ─────────]
요리사 3:                   [반찬 만들기 ──]
→ 실제로 3가지 요리를 "동시에" 진행
```

---

## 📊 시각적 비교

### 동시성 (단일 코어)

```
시간 ─────────────────────────────────>
CPU:  [A][B][A][C][B][A][C][B][C]
```

- 하나의 CPU가 작업 A, B, C를 **번갈아가며** 실행
- Context Switching으로 "동시에 실행되는 것처럼" 보임
- **실제로는 순차 실행**

### 병렬성 (멀티 코어)

```
시간 ─────────────────────────────────>
CPU1: [A][A][A][A][A][A][A]
CPU2: [B][B][B][B][B][B][B]
CPU3: [C][C][C][C][C][C][C]
```

- 여러 CPU가 작업을 **동시에** 실행
- **실제로 동시 실행**

### 동시성 + 병렬성

```
시간 ─────────────────────────────────>
CPU1: [A][B][A][C][A]
CPU2: [B][C][B][A][C]
CPU3: [C][A][C][B][B]
```

- 여러 CPU에서 작업을 번갈아가며 실행
- 현대 멀티코어 시스템의 일반적인 상황

---

## 📋 상세 비교표

| 특성 | 동시성 (Concurrency) | 병렬성 (Parallelism) |
|------|---------------------|---------------------|
| **정의** | 여러 작업을 다루는 구조 | 여러 작업을 동시 실행 |
| **목적** | 응답성, 처리량 향상 | 성능 향상 (속도) |
| **CPU 요구** | 단일 코어도 가능 | 멀티 코어 필수 |
| **실행 방식** | Time-slicing (번갈아가며) | 동시 실행 |
| **예시** | 웹 서버 (여러 요청 처리) | 병렬 계산 (행렬 곱셈) |
| **언어 지원** | async/await, Goroutine | OpenMP, CUDA |
| **복잡도** | 동기화 문제 (Race, Deadlock) | 데이터 분할, 동기화 |
| **확장성** | 스레드 수 증가 | 코어 수 증가 |

---

## 🎯 실전 예시

### 예시 1: 커피숍 (동시성)

**상황**: 바리스타 1명, 주문 3개

```
바리스타의 동시성 처리:
1. 주문 A 에스프레소 추출 시작 (30초 소요)
2. 기계가 작동하는 동안 주문 B 우유 스팀 (20초)
3. 다시 주문 A 에스프레소 완성
4. 주문 C 준비 시작
```

**관찰**:
- 바리스타 1명이지만 **동시에 여러 주문을 처리**
- I/O 대기 시간을 활용한 동시성

### 예시 2: 공장 조립 라인 (병렬성)

**상황**: 작업자 4명, 자동차 조립

```
작업자 1: [엔진 조립 ──────]
작업자 2:          [타이어 장착 ──────]
작업자 3:                   [도색 ──────]
작업자 4:                            [검수 ──]
```

**관찰**:
- 4명이 **동시에 서로 다른 작업**
- 물리적 병렬 실행

---

## 💻 코드 예시

### 동시성: I/O-bound (Node.js)

```javascript
// 비동기 I/O로 동시성 달성 (단일 스레드)
async function processRequests() {
    const p1 = fetch('https://api1.com/data');  // I/O 시작
    const p2 = fetch('https://api2.com/data');  // I/O 시작
    const p3 = fetch('https://api3.com/data');  // I/O 시작

    // 모든 I/O가 완료될 때까지 대기 (동시에 진행됨)
    const [r1, r2, r3] = await Promise.all([p1, p2, p3]);
    return [r1, r2, r3];
}
```

**특징**: 단일 스레드이지만 I/O 대기 시간에 다른 작업 처리

### 병렬성: CPU-bound (C++)

```cpp
#include <thread>
#include <vector>
#include <numeric>

// 대용량 배열을 병렬로 합산
long long parallel_sum(const std::vector<int>& data, int num_threads) {
    std::vector<std::thread> threads;
    std::vector<long long> results(num_threads, 0);

    int chunk_size = data.size() / num_threads;

    // 각 스레드가 배열의 일부를 처리
    for (int i = 0; i < num_threads; i++) {
        threads.emplace_back([&, i]() {
            int start = i * chunk_size;
            int end = (i == num_threads - 1) ? data.size() : start + chunk_size;
            results[i] = std::accumulate(data.begin() + start,
                                          data.begin() + end, 0LL);
        });
    }

    // 모든 스레드 완료 대기
    for (auto& t : threads) t.join();

    // 부분 결과 합산
    return std::accumulate(results.begin(), results.end(), 0LL);
}
```

**특징**: 실제로 여러 코어에서 동시 계산 → 속도 향상

---

## 📈 Amdahl의 법칙 (Amdahl's Law)

**질문**: 병렬화로 얼마나 빨라질 수 있을까?

**공식**:
```
Speedup = 1 / ((1 - P) + P/N)

P: 병렬화 가능한 부분 (0~1)
N: 프로세서 수
```

### 예시 계산

| 병렬화 비율 (P) | 2 코어 | 4 코어 | 8 코어 | 무한 코어 |
|----------------|--------|--------|--------|-----------|
| 50% | 1.33x | 1.60x | 1.78x | 2.00x |
| 75% | 1.60x | 2.29x | 2.91x | 4.00x |
| 90% | 1.82x | 3.08x | 4.71x | 10.00x |
| 95% | 1.90x | 3.48x | 5.93x | 20.00x |

**관찰**:
- 병렬화 비율이 낮으면 코어를 늘려도 효과 제한적
- 95% 병렬화해도 무한 코어로 20배까지만 빨라짐

**시각화**:
```
P = 50% (절반만 병렬화)
┌─────────────────┐
│  Serial  │Parallel│
│   50%    │  50%  │
└─────────────────┘

4 코어로 실행:
┌─────────────────┐
│ Serial │ P/4 │   ← Speedup: 1 / (0.5 + 0.5/4) = 1.6x
│  50%   │12.5%│
└─────────────────┘
```

---

## 🚀 Gustafson의 법칙

**Amdahl의 한계**: 문제 크기가 고정되어 있다고 가정

**Gustafson의 관점**: 코어가 많으면 **더 큰 문제**를 풀 수 있다

**공식**:
```
Speedup = N - (1 - P) × (N - 1)

P: 병렬 실행 시간 비율
N: 프로세서 수
```

**예시**:
- 4 코어, 병렬 비율 75%
- Amdahl: 2.29배 빠름 (고정된 문제)
- Gustafson: 3.25배 큰 문제 해결 가능

**적용**:
- 빅데이터 분석 (데이터가 많을수록 병렬화 효과)
- 렌더링 (해상도 높이기)
- 시뮬레이션 (정밀도 향상)

---

## 🎓 CPU-bound vs I/O-bound

### CPU-bound 작업

**정의**: CPU 계산이 병목인 작업

**특징**:
- 계산 집약적 (암호화, 압축, 렌더링)
- **병렬성**으로 성능 향상
- 최적 스레드 수 = CPU 코어 수

**예시**:
```cpp
// CPU-bound: 소수 찾기
bool is_prime(int n) {
    for (int i = 2; i * i <= n; i++) {
        if (n % i == 0) return false;
    }
    return true;
}

// 병렬화로 속도 향상
void find_primes_parallel(int start, int end, int num_threads) {
    // 범위를 num_threads로 분할하여 병렬 처리
}
```

### I/O-bound 작업

**정의**: I/O 대기 시간이 병목인 작업

**특징**:
- 디스크, 네트워크, 데이터베이스 접근
- **동시성**으로 처리량 향상
- 최적 스레드 수 = 코어 수 × (1 + 대기시간/CPU시간)

**예시**:
```javascript
// I/O-bound: 웹 API 호출
async function fetchMultipleAPIs() {
    // 비동기로 동시 요청 (단일 스레드로도 가능)
    const results = await Promise.all([
        fetch('https://api1.com'),
        fetch('https://api2.com'),
        fetch('https://api3.com')
    ]);
    return results;
}
```

---

## 🔍 동시성 패러다임

### 1. 공유 메모리 (Threads)

**모델**: 여러 스레드가 같은 메모리 공유

**예시**: C++, Java, C#

```cpp
std::mutex mtx;
int shared_data = 0;

void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    shared_data++;
}
```

**장점**: 빠른 통신
**단점**: 동기화 복잡, Race Condition

### 2. 메시지 패싱 (Channels)

**모델**: 독립된 프로세스/스레드가 메시지로 통신

**예시**: Go, Erlang

```go
ch := make(chan int)

go func() {
    ch <- 42  // 메시지 전송
}()

result := <-ch  // 메시지 수신
```

**장점**: Race Condition 방지
**단점**: 메시지 전달 오버헤드

### 3. Actor 모델

**모델**: 각 Actor가 독립적으로 메시지 처리

**예시**: Akka, Erlang

```scala
class MyActor extends Actor {
    def receive = {
        case msg: String => println(s"Received: $msg")
    }
}
```

**장점**: 완전한 격리, 확장성
**단점**: 복잡한 메시지 흐름

---

## 🎯 언제 무엇을 사용할까?

### 동시성이 필요한 경우

✅ **I/O-bound 작업**
- 웹 서버 (수천 개의 동시 연결)
- 데이터베이스 쿼리
- 네트워크 통신

✅ **응답성 향상**
- UI 블로킹 방지
- 백그라운드 작업

**도구**: async/await, Event Loop, Goroutines

### 병렬성이 필요한 경우

✅ **CPU-bound 작업**
- 과학 계산
- 이미지/비디오 처리
- 머신러닝 학습

✅ **대용량 데이터 처리**
- 빅데이터 분석
- 병렬 정렬

**도구**: Thread Pool, OpenMP, CUDA

---

## 📊 언어별 접근 방식

| 언어 | 동시성 | 병렬성 |
|------|--------|--------|
| **JavaScript** | Event Loop, async/await | Web Workers |
| **Python** | asyncio, threading | multiprocessing, Numba |
| **Go** | Goroutines, Channels | GOMAXPROCS, 병렬 루프 |
| **C++** | std::async | std::thread, OpenMP |
| **Java** | CompletableFuture | Parallel Streams, ForkJoin |
| **Rust** | async/await, tokio | Rayon |

---

## 🔗 다음 단계

- [스레드 생명주기](./03-thread-lifecycle.md) - 스레드 상태 이해
- [컨텍스트 스위칭](./04-context-switching.md) - 동시성의 비용
- [동시성 패턴](../04-concurrency-patterns/README.md) - 실전 패턴

---

## 📚 참고 자료

- [Concurrency is not Parallelism (Rob Pike)](https://go.dev/blog/waza-talk)
- "Seven Concurrency Models in Seven Weeks" - Paul Butcher

---

*동시성과 병렬성을 이해하면 적절한 도구와 패턴을 선택할 수 있습니다!*
