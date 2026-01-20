# 추천 레포지토리

## 📌 개요

멀티스레드 프로그래밍 학습에 도움이 되는 추천 오픈소스 레포지토리 목록입니다. 언어별, 난이도별로 분류하여 제시합니다.

---

## 🎯 선정 기준

- **학습 가치**: 명확한 패턴과 베스트 프랙티스
- **코드 품질**: 읽기 쉽고 잘 문서화됨
- **활성도**: 활발히 유지보수됨
- **실용성**: 실제 프로덕션에서 사용 가능
- **테스트**: 충분한 테스트 커버리지

---

## C/C++ 라이브러리

### 1. Concurrency Primitives

#### libcds (⭐⭐⭐⭐⭐)
```
Repository: https://github.com/khizmax/libcds
Stars: 2.5k+
License: Boost Software License
```

**특징:**
- Lock-Free/Wait-Free 자료구조
- 다양한 메모리 회수 기법 (Hazard Pointers, EBR)
- Header-only 라이브러리

**학습 포인트:**
- Lock-Free 알고리즘 구현
- 메모리 회수 패턴
- CAS 연산 활용

**추천 대상:** 고급 개발자

---

#### Facebook Folly (⭐⭐⭐⭐)
```
Repository: https://github.com/facebook/folly
Stars: 27k+
License: Apache 2.0
```

**특징:**
- 프로덕션 레벨 C++ 컴포넌트
- MPMCQueue, Future/Promise
- ThreadPoolExecutor

**학습 포인트:**
- 엔터프라이즈급 패턴
- 비동기 프로그래밍
- 성능 최적화

**추천 대상:** 중급~고급 개발자

---

#### Boost.Lockfree (⭐⭐⭐)
```
Repository: https://github.com/boostorg/lockfree
Part of: Boost C++ Libraries
License: Boost Software License
```

**특징:**
- Lock-Free Queue, Stack
- Boost 라이브러리의 일부
- 잘 문서화됨

**학습 포인트:**
- 표준 라이브러리 수준의 코드 품질
- Lock-Free 자료구조 기초

**추천 대상:** 중급 개발자

---

#### ConcurrencyKit (⭐⭐⭐)
```
Repository: https://github.com/concurrencykit/ck
Stars: 2.3k+
License: BSD
```

**특징:**
- C 언어 Lock-Free 라이브러리
- 다양한 플랫폼 지원
- 성능 중심 설계

**학습 포인트:**
- 저수준 동시성 프리미티브
- 플랫폼별 최적화

**추천 대상:** 시스템 프로그래머

---

### 2. Threading Libraries

#### Intel TBB (Threading Building Blocks) (⭐⭐⭐⭐)
```
Repository: https://github.com/oneapi-src/oneTBB
Stars: 5.6k+
License: Apache 2.0
```

**특징:**
- Task-based 병렬 프로그래밍
- 병렬 알고리즘 (parallel_for, parallel_reduce)
- Concurrent 컨테이너

**학습 포인트:**
- Work-stealing 알고리즘
- Task 기반 동시성
- 고수준 추상화

**추천 대상:** 중급 개발자

---

#### HPX (High Performance ParalleX) (⭐⭐⭐⭐)
```
Repository: https://github.com/STEllAR-GROUP/hpx
Stars: 2.5k+
License: Boost Software License
```

**특징:**
- C++ 표준 라이브러리의 병렬 버전
- 분산 컴퓨팅 지원
- Future/Promise 기반

**학습 포인트:**
- C++17/20 병렬 알고리즘
- 분산 시스템 패턴

**추천 대상:** 고급 개발자

---

## Go 라이브러리

### 1. Concurrency Primitives

#### xsync (⭐⭐⭐)
```
Repository: https://github.com/puzpuzpuz/xsync
Stars: 3k+
License: Apache 2.0
```

**특징:**
- 제네릭 기반 concurrent 자료구조
- MapOf, MPMCQueue, Counter
- 고성능

**학습 포인트:**
- Go 제네릭 활용
- Lock-Free 구현

**추천 대상:** 중급 Go 개발자

---

#### workerpool (⭐⭐)
```
Repository: https://github.com/gammazero/workerpool
Stars: 1.2k+
License: MIT
```

**특징:**
- 간단한 Worker Pool 구현
- 동적 크기 조정
- 사용하기 쉬운 API

**학습 포인트:**
- Worker Pool 패턴
- Goroutine 관리

**추천 대상:** 초급~중급 Go 개발자

---

#### conc (⭐⭐⭐)
```
Repository: https://github.com/sourcegraph/conc
Stars: 8k+
License: MIT
```

**특징:**
- 더 나은 Go 동시성 프리미티브
- WaitGroup, Pool, Stream
- 에러 처리 개선

**학습 포인트:**
- 안전한 동시성 패턴
- 에러 전파

**추천 대상:** 중급 Go 개발자

---

### 2. Game Servers

#### Nakama (⭐⭐⭐)
```
Repository: https://github.com/heroiclabs/nakama
Stars: 8.5k+
License: Apache 2.0
```

**특징:**
- 완전한 게임 서버 프레임워크
- Goroutine 기반 동시성
- 매치메이킹, 리더보드 등

**학습 포인트:**
- Go 동시성 패턴
- 게임 서버 아키텍처

**추천 대상:** 중급 개발자

---

#### Pitaya (⭐⭐⭐)
```
Repository: https://github.com/topfreegames/pitaya
Stars: 2.2k+
License: MIT
```

**특징:**
- 확장 가능한 게임 서버
- Actor 모델
- 클러스터링 지원

**학습 포인트:**
- Actor 패턴
- 분산 시스템

**추천 대상:** 중급~고급 개발자

---

## Rust 라이브러리

### 1. Concurrency

#### crossbeam (⭐⭐⭐⭐)
```
Repository: https://github.com/crossbeam-rs/crossbeam
Stars: 6.8k+
License: Apache 2.0/MIT
```

**특징:**
- Lock-Free 자료구조
- Channel, Epoch-based GC
- 안전한 동시성

**학습 포인트:**
- Rust의 동시성 모델
- Ownership과 동시성

**추천 대상:** 중급 Rust 개발자

---

#### rayon (⭐⭐⭐)
```
Repository: https://github.com/rayon-rs/rayon
Stars: 10k+
License: Apache 2.0/MIT
```

**특징:**
- 데이터 병렬 처리
- Work-stealing 스케줄러
- 간단한 API

**학습 포인트:**
- 병렬 이터레이터
- Work-stealing

**추천 대상:** 초급~중급 Rust 개발자

---

#### tokio (⭐⭐⭐⭐)
```
Repository: https://github.com/tokio-rs/tokio
Stars: 25k+
License: MIT
```

**특징:**
- 비동기 런타임
- async/await 지원
- 고성능 I/O

**학습 포인트:**
- 비동기 프로그래밍
- Future/Stream

**추천 대상:** 중급 Rust 개발자

---

## JavaScript/TypeScript

### 1. Game Servers

#### Colyseus (⭐⭐)
```
Repository: https://github.com/colyseus/colyseus
Stars: 7.1k+
License: MIT
```

**특징:**
- 멀티플레이어 게임 서버
- 자동 상태 동기화
- Room 기반 아키텍처

**학습 포인트:**
- Event Loop 동시성
- 상태 동기화

**추천 대상:** 초급~중급 개발자

---

#### Socket.IO (⭐⭐⭐)
```
Repository: https://github.com/socketio/socket.io
Stars: 60k+
License: MIT
```

**특징:**
- 실시간 양방향 통신
- WebSocket 기반
- 자동 재연결

**학습 포인트:**
- 실시간 통신 패턴
- Event-driven 아키텍처

**추천 대상:** 초급~중급 개발자

---

### 2. Worker Threads

#### piscina (⭐⭐)
```
Repository: https://github.com/piscinajs/piscina
Stars: 4k+
License: MIT
```

**특징:**
- Worker Thread Pool
- Node.js Worker Threads 활용
- TypeScript 지원

**학습 포인트:**
- Node.js 멀티스레딩
- Worker Thread 패턴

**추천 대상:** 중급 Node.js 개발자

---

## Java

### 1. Concurrency Utilities

#### JCTools (⭐⭐⭐⭐)
```
Repository: https://github.com/JCTools/JCTools
Stars: 3.5k+
License: Apache 2.0
```

**특징:**
- Lock-Free Queue
- Padding 최적화
- 고성능 concurrent 자료구조

**학습 포인트:**
- Java Lock-Free 구현
- False Sharing 회피

**추천 대상:** 중급~고급 Java 개발자

---

#### Disruptor (⭐⭐⭐⭐)
```
Repository: https://github.com/LMAX-Exchange/disruptor
Stars: 17k+
License: Apache 2.0
```

**특징:**
- Ring Buffer 기반 메시지 큐
- 극도로 낮은 레이턴시
- Lock-Free 설계

**학습 포인트:**
- Ring Buffer 패턴
- 메모리 배리어

**추천 대상:** 고급 Java 개발자

---

## Python

### 1. Async/Concurrent

#### asyncio (표준 라이브러리)
```
Documentation: https://docs.python.org/3/library/asyncio.html
Part of: Python Standard Library
```

**특징:**
- 비동기 I/O 프레임워크
- async/await 지원
- Event Loop

**학습 포인트:**
- Python 비동기 프로그래밍
- Coroutine

**추천 대상:** 모든 Python 개발자

---

#### multiprocessing (표준 라이브러리)
```
Documentation: https://docs.python.org/3/library/multiprocessing.html
Part of: Python Standard Library
```

**특징:**
- 진정한 병렬 처리 (GIL 우회)
- Process Pool
- 공유 메모리

**학습 포인트:**
- 프로세스 기반 병렬 처리
- IPC (Inter-Process Communication)

**추천 대상:** 중급 Python 개발자

---

## 학습 로드맵

### 초급자
1. **Colyseus** - Event-driven 동시성
2. **workerpool (Go)** - Worker Pool 패턴
3. **asyncio (Python)** - 비동기 프로그래밍

### 중급자
1. **Folly** - 프로덕션 패턴
2. **xsync** - Lock-Free 자료구조
3. **crossbeam** - 안전한 동시성
4. **TBB** - Task 기반 병렬 처리

### 고급자
1. **libcds** - 고급 Lock-Free 알고리즘
2. **Disruptor** - 극한의 성능 최적화
3. **HPX** - 분산 병렬 처리

---

## 카테고리별 추천

### Lock-Free 자료구조
1. **libcds** (C++) - 가장 포괄적
2. **crossbeam** (Rust) - 안전성과 성능
3. **JCTools** (Java) - 실용적

### 게임 서버
1. **Nakama** (Go) - 완전한 프레임워크
2. **Colyseus** (TS) - 빠른 프로토타이핑
3. **Pitaya** (Go) - 확장 가능

### 비동기 프로그래밍
1. **tokio** (Rust) - 최고 성능
2. **Folly Future** (C++) - 엔터프라이즈급
3. **asyncio** (Python) - 접근성

### Task 기반 병렬 처리
1. **TBB** (C++) - 업계 표준
2. **rayon** (Rust) - 간단함
3. **HPX** (C++) - 분산 환경

---

## 실습 프로젝트 아이디어

### 1. Lock-Free 자료구조 구현
- libcds를 참고하여 직접 구현
- 벤치마크로 성능 비교

### 2. 게임 서버 구축
- Nakama 또는 Colyseus 활용
- 간단한 멀티플레이어 게임 제작

### 3. Worker Pool 라이브러리
- xsync를 참고하여 자신만의 구현
- 다양한 언어로 포팅

### 4. 비동기 HTTP 서버
- Folly 또는 tokio 활용
- 고성능 웹 서버 구현

---

## 추가 리소스

### 컨퍼런스 발표
- **CppCon**: C++ 동시성 최신 기술
- **GopherCon**: Go 동시성 패턴
- **RustConf**: Rust 안전한 동시성

### 온라인 코스
- MIT 6.824: Distributed Systems
- Coursera: Parallel Programming in Java
- Udacity: High Performance Computing

### 책
- "C++ Concurrency in Action" (Anthony Williams)
- "The Art of Multiprocessor Programming" (Herlihy & Shavit)
- "Concurrency in Go" (Katherine Cox-Buday)

---

## 기여 방법

오픈소스에 기여하며 학습하는 방법:

1. **이슈 해결**: Good First Issue 찾기
2. **문서 개선**: 오타 수정, 예제 추가
3. **테스트 작성**: 커버리지 향상
4. **벤치마크**: 성능 측정 및 최적화
5. **버그 리포트**: 재현 가능한 버그 제보

---

## 마무리

각 레포지토리는 독특한 학습 가치를 제공합니다. 자신의 수준과 관심사에 맞는 프로젝트를 선택하여 깊이 있게 분석하고, 실제로 사용해보며 학습하시기 바랍니다.

**추천 학습 방법:**
1. 소스 코드 읽기
2. 예제 실행
3. 벤치마크 측정
4. 직접 수정해보기
5. 프로젝트에 적용

---

*이 목록은 지속적으로 업데이트됩니다. 추천할 레포지토리가 있다면 기여해주세요!*
