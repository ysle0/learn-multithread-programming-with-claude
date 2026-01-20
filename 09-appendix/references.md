# 참고 자료 및 리소스

## 📌 개요

멀티스레드 프로그래밍 학습을 위한 포괄적인 참고 자료 모음입니다.

---

## 📚 필독 도서

### 입문 ~ 중급

#### 1. "C++ Concurrency in Action" (2nd Edition)
- **저자**: Anthony Williams
- **출판**: Manning, 2019
- **언어**: C++11/14/17
- **난이도**: ⭐⭐⭐

**내용:**
- C++ 표준 스레드 라이브러리
- 메모리 모델 상세 설명
- Lock-Free 프로그래밍
- 디자인 패턴

**추천 이유:**
- C++ 동시성의 바이블
- 실전 예제 풍부
- 최신 표준 반영

---

#### 2. "The Art of Multiprocessor Programming"
- **저자**: Maurice Herlihy, Nir Shavit
- **출판**: Morgan Kaufmann, 2020 (2nd Edition)
- **언어**: Java (개념은 언어 무관)
- **난이도**: ⭐⭐⭐⭐

**내용:**
- 동시성 이론 기초
- Lock-Free 알고리즘
- Progress 보장 (Wait-Free, Lock-Free)
- 정확성 증명

**추천 이유:**
- 이론적 기초 탄탄
- 수학적 증명 포함
- 학술적으로 엄밀

---

#### 3. "Java Concurrency in Practice"
- **저자**: Brian Goetz et al.
- **출판**: Addison-Wesley, 2006
- **언어**: Java
- **난이도**: ⭐⭐⭐

**내용:**
- Java 동시성 API
- Thread-safe 설계
- 성능과 확장성
- 실전 패턴

**추천 이유:**
- Java 개발자 필독서
- 원칙과 패턴 중심
- 시간이 지나도 유효

---

#### 4. "Concurrency in Go"
- **저자**: Katherine Cox-Buday
- **출판**: O'Reilly, 2017
- **언어**: Go
- **난이도**: ⭐⭐

**내용:**
- Goroutine과 Channel
- 동시성 패턴
- 에러 처리
- 성능 최적화

**추천 이유:**
- Go 동시성 완벽 가이드
- 실용적 예제
- 베스트 프랙티스

---

### 고급

#### 5. "Is Parallel Programming Hard, And, If So, What Can You Do About It?"
- **저자**: Paul E. McKenney
- **출판**: kernel.org (무료)
- **링크**: https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html
- **난이도**: ⭐⭐⭐⭐⭐

**내용:**
- Linux 커널 동시성
- RCU (Read-Copy-Update)
- 메모리 배리어
- Lock-Free 알고리즘

**추천 이유:**
- 무료!
- 리눅스 커널 개발자 작성
- 실무 경험 기반

---

#### 6. "Programming with POSIX Threads"
- **저자**: David R. Butenhof
- **출판**: Addison-Wesley, 1997
- **언어**: C (POSIX)
- **난이도**: ⭐⭐⭐⭐

**내용:**
- POSIX Threads API
- 저수준 동기화
- 디버깅과 성능
- 이식성

**추천 이유:**
- POSIX Threads 표준 문서
- Unix/Linux 기초
- 저수준 이해

---

## 📖 학술 논문

### 필독 논문

#### Lock-Free 자료구조

1. **"Simple, Fast, and Practical Non-Blocking and Blocking Concurrent Queue Algorithms"**
   - Maged M. Michael, Michael L. Scott (1996)
   - 링크: https://www.cs.rochester.edu/u/scott/papers/1996_PODC_queues.pdf
   - Michael-Scott Queue의 원조

2. **"Hazard Pointers: Safe Memory Reclamation for Lock-Free Objects"**
   - Maged M. Michael (2004)
   - 메모리 회수 기법의 표준

3. **"Split-Ordered Lists: Lock-Free Extensible Hash Tables"**
   - Ori Shalev, Nir Shavit (2006)
   - 동적 크기 조정 해시 맵

#### 메모리 모델

4. **"A Primer on Memory Consistency and Cache Coherence"**
   - Daniel J. Sorin et al. (2011)
   - 메모리 일관성 모델 입문

5. **"Mathematizing C++ Concurrency"**
   - Mark Batty et al. (2011)
   - C++11 메모리 모델 형식화

#### 검증과 테스팅

6. **"Finding and Reproducing Heisenbugs in Concurrent Programs"**
   - Madanlal Musuvathi et al. (2008)
   - 동시성 버그 재현

---

## 🎓 온라인 코스

### 무료 코스

#### 1. MIT 6.824: Distributed Systems
- **강사**: Robert Morris
- **링크**: https://pdos.csail.mit.edu/6.824/
- **난이도**: ⭐⭐⭐⭐

**내용:**
- 분산 시스템 기초
- 일관성과 복제
- Raft 합의 알고리즘
- MapReduce

---

#### 2. Coursera: Parallel Programming in Java
- **대학**: Rice University
- **강사**: Vivek Sarkar
- **난이도**: ⭐⭐⭐

**내용:**
- Fork-Join 프레임워크
- Stream API
- 병렬 알고리즘

---

#### 3. Udacity: High Performance Computing
- **난이도**: ⭐⭐⭐⭐

**내용:**
- GPU 프로그래밍
- CUDA
- 병렬 알고리즘

---

### 유료 코스

#### Pluralsight: C++ Concurrency
- 실전 C++ 동시성
- 디자인 패턴
- 프로덕션 코드

---

## 🎥 컨퍼런스 발표

### CppCon (C++)

#### 추천 발표

1. **"Lock-Free Programming (or, Juggling Razor Blades)"**
   - Herb Sutter (2014)
   - 링크: https://www.youtube.com/watch?v=c1gO9aB9nbs

2. **"atomic<> Weapons"**
   - Herb Sutter (2012)
   - C++11 atomic 완벽 가이드

3. **"The C++ Memory Model"**
   - Valentin Ziegler (2019)
   - 메모리 모델 심층 분석

---

### GopherCon (Go)

#### 추천 발표

1. **"Concurrency is Not Parallelism"**
   - Rob Pike (2012)
   - 동시성 vs 병렬성

2. **"Go Concurrency Patterns"**
   - Rob Pike (2012)
   - 기본 패턴 소개

3. **"Advanced Go Concurrency Patterns"**
   - Sameer Ajmani (2013)
   - 고급 패턴

---

### RustConf (Rust)

1. **"Fearless Concurrency"**
   - Aaron Turon (2015)
   - Rust 동시성 철학

2. **"Lock-free Rust: Crossbeam"**
   - Aaron Turon (2017)

---

## 🌐 웹 리소스

### 공식 문서

#### C++
- cppreference.com: https://en.cppreference.com/w/cpp/thread
- C++ 표준 초안: https://eel.is/c++draft/

#### Go
- The Go Blog: https://go.dev/blog/
- Effective Go: https://go.dev/doc/effective_go

#### Rust
- The Rust Book: https://doc.rust-lang.org/book/
- Rustonomicon: https://doc.rust-lang.org/nomicon/

---

### 블로그

#### 1. Preshing on Programming
- **저자**: Jeff Preshing
- **링크**: https://preshing.com/
- **주제**: Lock-Free, Memory Ordering

**추천 글:**
- "An Introduction to Lock-Free Programming"
- "Memory Ordering at Compile Time"
- "The Happens-Before Relation"

---

#### 2. 1024cores
- **저자**: Dmitry Vyukov
- **링크**: http://www.1024cores.net/
- **주제**: Lock-Free 알고리즘, 동시성 패턴

**추천 글:**
- "Lockfree Algorithms"
- "Memory Models"
- "Bounded MPMC Queue"

---

#### 3. Mechanical Sympathy
- **저자**: Martin Thompson
- **링크**: https://mechanical-sympathy.blogspot.com/
- **주제**: 하드웨어와 소프트웨어 상호작용

---

### Stack Overflow 태그

- [c++] + [multithreading]
- [concurrency]
- [lock-free]
- [goroutine]
- [rust] + [concurrency]

---

## 🛠️ 도구 및 라이브러리

### 분석 도구
- **Valgrind**: https://valgrind.org/
- **ThreadSanitizer**: https://github.com/google/sanitizers
- **Intel Inspector**: https://www.intel.com/content/www/us/en/developer/tools/oneapi/inspector.html

### 벤치마킹
- **Google Benchmark**: https://github.com/google/benchmark
- **Criterion (Rust)**: https://github.com/bheisler/criterion.rs
- **JMH (Java)**: https://github.com/openjdk/jmh

### 테스팅
- **Google Test**: https://github.com/google/googletest
- **Catch2**: https://github.com/catchorg/Catch2

---

## 📰 뉴스레터 및 팟캐스트

### 뉴스레터
- **C++ Weekly**: Jason Turner
- **Go Weekly**: Golang 뉴스
- **This Week in Rust**

### 팟캐스트
- **CppCast**: C++ 팟캐스트
- **Go Time**: Go 팟캐스트

---

## 🎮 실습 프로젝트 아이디어

### 초급
1. **Thread-Safe Queue**: Producer-Consumer 패턴
2. **Thread Pool**: 작업 스케줄러
3. **Rate Limiter**: Token Bucket 알고리즘

### 중급
1. **Cache 시스템**: LRU Cache with TTL
2. **Task Scheduler**: 우선순위 기반
3. **Web Crawler**: 멀티스레드 크롤러

### 고급
1. **Lock-Free Queue**: Michael-Scott Queue 구현
2. **In-Memory Database**: MVCC 구현
3. **분산 Key-Value Store**: Raft 합의 알고리즘

---

## 📊 벤치마크 레포지토리

1. **concurrency-benchmarks**
   - https://github.com/preshing/concurrency-benchmarks
   - 다양한 동시성 프리미티브 비교

2. **lock-free-benchmarks**
   - Lock-Free vs Lock-Based 성능 비교

---

## 🔍 검색 키워드

### 영어
- "lock-free programming"
- "memory ordering"
- "happens-before relationship"
- "ABA problem"
- "wait-free algorithm"
- "compare-and-swap"
- "memory barrier"

### 한국어
- "멀티스레딩"
- "동시성 프로그래밍"
- "락프리 알고리즘"
- "메모리 순서"

---

## 💬 커뮤니티

### Reddit
- r/cpp
- r/golang
- r/rust
- r/concurrency

### Discord
- C++ Discord
- Rust Programming Language Community
- Gophers Slack

### Stack Overflow
- 태그별 질문/답변 활발

---

## 📅 학습 로드맵

### 1개월 계획 (기초)
- **Week 1**: 기본 개념 (Thread, Mutex)
- **Week 2**: 동기화 프리미티브
- **Week 3**: 패턴 (Producer-Consumer, Reader-Writer)
- **Week 4**: 실습 프로젝트

### 3개월 계획 (중급)
- **Month 1**: 기초 복습 및 심화
- **Month 2**: Lock-Free 프로그래밍 입문
- **Month 3**: 실전 프로젝트 (Thread Pool, Cache)

### 6개월 계획 (고급)
- **Month 1-2**: 기초 및 중급 마스터
- **Month 3-4**: Lock-Free 알고리즘 구현
- **Month 5-6**: 분산 시스템 및 고급 주제

---

## 🎯 추천 학습 순서

### C++ 개발자
1. "C++ Concurrency in Action"
2. CppCon 발표 시청
3. libcds 소스 코드 분석
4. Lock-Free 자료구조 직접 구현

### Go 개발자
1. "Concurrency in Go"
2. Go Blog 읽기
3. Nakama 소스 코드 분석
4. 실전 프로젝트 (게임 서버)

### Rust 개발자
1. "The Rust Book" Concurrency 챕터
2. crossbeam 소스 코드 분석
3. tokio 비동기 프로그래밍
4. Lock-Free 자료구조 구현

---

## 📖 정기 업데이트 자료

이 문서는 정기적으로 업데이트됩니다:
- 새로운 도서 출간
- 최신 컨퍼런스 발표
- 유용한 블로그 포스트
- 오픈소스 프로젝트

---

## 🤝 기여

이 학습 자료에 추가하고 싶은 리소스가 있다면:
1. GitHub Issue 생성
2. Pull Request 제출
3. 커뮤니티 토론

---

## ⭐ 즐겨찾기 추천

반드시 북마크할 사이트:
1. cppreference.com (C++)
2. go.dev (Go)
3. doc.rust-lang.org (Rust)
4. preshing.com (Lock-Free)
5. 1024cores.net (동시성)

---

## 📝 마무리

멀티스레드 프로그래밍은 지속적인 학습이 필요한 분야입니다. 이 문서의 자료들을 참고하여 단계적으로 학습하시기 바랍니다.

**학습 원칙:**
1. 이론과 실습의 균형
2. 작은 프로젝트부터 시작
3. 코드 읽기와 분석
4. 커뮤니티 참여
5. 지속적인 연습

**행운을 빕니다!**

---

*이 학습 레포지토리가 도움이 되었다면 GitHub Star를 부탁드립니다!*
