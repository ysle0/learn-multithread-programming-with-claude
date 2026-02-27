# C++ 동시성

C++는 C++11부터 저수준, 고성능 동시성 프리미티브를 제공합니다. 이 언어는 "제로 오버헤드 추상화" 철학을 따르며, 안전하고 현대적인 추상화를 제공하면서 최대한의 제어권을 줍니다.

## 개요

C++ 동시성은 크게 발전해 왔습니다:
- **C++11**: 스레딩 라이브러리, atomic, 메모리 모델 도입
- **C++14**: 소규모 개선 및 버그 수정
- **C++17**: 병렬 알고리즘
- **C++20**: 코루틴, 세마포어, 배리어, 래치, jthread
- **C++23**: 동기화 프리미티브의 추가 개선

## 핵심 구성 요소

### 1. [std::thread](./01-std-thread.md)
- OS 스레드 생성 및 관리
- 스레드 생명주기와 조인
- 스레드에 인수 전달
- 스레드 ID와 하드웨어 동시성

### 2. [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
- 상호 배제를 위한 `std::mutex`
- RAII 락 가드 (`std::lock_guard`, `std::unique_lock`)
- Reader-Writer 시나리오를 위한 공유 뮤텍스
- 재귀 및 시간 제한 뮤텍스

### 3. [Atomic 연산](./03-atomic.md)
- Lock-Free 프로그래밍을 위한 `std::atomic<T>`
- 메모리 순서와 동기화
- Compare-and-Swap 연산
- Atomic 스마트 포인터 (C++20)

### 4. [Condition Variable](./04-condition-variable.md)
- 조건이 참이 될 때까지 대기
- 생산자-소비자 패턴
- 가짜 깨어남(Spurious Wakeup)과 조건 루프
- notify_one vs. notify_all

### 5. [Async와 Future](./05-async-future.md)
- `std::async`를 이용한 태스크 기반 병렬처리
- `std::future`와 `std::promise`
- 실행 정책 (async vs. deferred)
- shared_future와 packaged_task

### 6. [Coroutine (C++20)](./06-coroutine.md)
- `co_await`, `co_yield`, `co_return` 키워드
- Promise Type과 Awaiter 구현
- Generator 패턴과 비동기 Task
- Symmetric Transfer와 HALO 최적화

## 다른 언어와의 비교

| 기능 | C++ | 비교 |
|---------|-----|------------|
| **스레드 생성** | `std::thread` | C# `Thread`와 유사, Go goroutine보다 무거움 |
| **비동기 패턴** | `std::async` + `std::future` | C# `async/await`나 JS Promise보다 사용이 덜 편리함 |
| **메시지 전달** | 내장 기능 없음 | Go 채널과 다름; 라이브러리 또는 수동 큐 사용 |
| **메모리 모델** | 잘 정의됨 (C++11) | 네 언어 중 가장 명시적이고 저수준 |
| **안전성** | 수동, 오류 발생 가능성 높음 | C#/Go/JS보다 덜 안전; 세심한 프로그래밍 필요 |

## 핵심 원칙

### 1. RAII (Resource Acquisition Is Initialization)
```cpp
{
    std::lock_guard<std::mutex> lock(mutex);
    // 임계 영역 - 스코프가 끝나면 자동으로 잠금 해제
}
```

### 2. 제로 오버헤드 추상화
C++ 스레딩 프리미티브는 최소한의 런타임 오버헤드로 효율적인 기계어 코드로 컴파일됩니다.

### 3. 명시적 메모리 순서
메모리 연산의 동기화 방식을 정확히 제어할 수 있습니다:
```cpp
atomic_var.store(value, std::memory_order_release);
auto val = atomic_var.load(std::memory_order_acquire);
```

## 일반적인 패턴

### 스레드 안전 싱글턴
```cpp
class Singleton {
    static Singleton& getInstance() {
        static Singleton instance;  // C++11 이상에서 스레드 안전
        return instance;
    }
};
```

### 범위 기반 잠금
```cpp
std::mutex m1, m2;
{
    std::scoped_lock lock(m1, m2);  // C++17: 두 뮤텍스를 잠그며 데드락 방지
    // 임계 영역
}
```

## 모범 사례

1. **고수준 추상화 선호**: 가능하면 수동 스레드 대신 `std::async` 사용
2. **RAII 사용**: 항상 lock guard를 사용하고, 수동으로 lock/unlock 하지 않기
3. **공유 상태 피하기**: 스레드 간 공유를 최소화
4. **const 정확성**: const 데이터는 안전하게 공유 가능
5. **Atomic 신중하게 사용**: relaxed atomic을 사용하기 전에 메모리 순서를 이해하기
6. **스레드 안전성 문서화**: 어떤 함수가 스레드 안전한지 표시
7. **값 의미론 선호**: 가능하면 이동 의미론과 함께 값으로 전달

## 일반적인 실수

### 1. join 또는 detach 잊기
```cpp
// 나쁨: 스레드 소멸자가 std::terminate를 호출함
void bad_example() {
    std::thread t([] { /* 작업 */ });
}  // 이런! join이나 detach를 안 했음

// 좋음: jthread (C++20) 사용 또는 join/detach 보장
void good_example() {
    std::jthread t([] { /* 작업 */ });
}  // 자동으로 join
```

### 2. 여러 Mutex로 인한 데드락
```cpp
// 나쁨: 데드락 발생 가능
mutex1.lock();
mutex2.lock();

// 좋음: scoped_lock 사용
std::scoped_lock lock(mutex1, mutex2);
```

### 3. 공유 데이터의 데이터 경쟁
```cpp
// 나쁨: 데이터 경쟁
int counter = 0;
std::thread t1([&] { ++counter; });
std::thread t2([&] { ++counter; });

// 좋음: atomic 또는 mutex 사용
std::atomic<int> counter{0};
std::thread t1([&] { ++counter; });
std::thread t2([&] { ++counter; });
```

### 4. 예외 안전성
```cpp
// 나쁨: 예외 발생 시 잠금이 해제되지 않음
mutex.lock();
might_throw();  // 잠금이 영원히 해제되지 않음!
mutex.unlock();

// 좋음: RAII 사용
{
    std::lock_guard lock(mutex);
    might_throw();  // 예외가 발생해도 잠금 해제됨
}
```

### 5. Condition Variable의 가짜 깨어남
```cpp
// 나쁨: 조건이 충족되지 않았는데 깨어날 수 있음
cv.wait(lock);
process_data();

// 좋음: 조건 서술어 사용
cv.wait(lock, [] { return data_ready; });
process_data();
```

## 성능 고려사항

### 스레드 생성 오버헤드
- 스레드 생성: ~100 마이크로초
- 컨텍스트 스위치: ~1-10 마이크로초
- 메모리 오버헤드: 스레드당 ~2MB (스택 크기)

**시사점**: 단기 작업을 위해 스레드를 생성하지 말고 스레드 풀을 사용하세요.

### 잠금 경합
- 비경합 잠금: ~25 나노초
- 경합 잠금: 1000배 더 느릴 수 있음

**시사점**: 임계 영역에서의 시간을 최소화하세요.

### False Sharing
```cpp
// 나쁨: False sharing - 카운터들이 같은 캐시 라인에 있음
struct Counters {
    std::atomic<int> counter1;
    std::atomic<int> counter2;
};

// 좋음: 정렬로 False sharing 방지
struct Counters {
    alignas(64) std::atomic<int> counter1;
    alignas(64) std::atomic<int> counter2;
};
```

## 현대 C++ 기능 (C++20 이후)

### jthread (조인 가능 스레드)
```cpp
std::jthread t([] {
    // 작업
});  // 소멸 시 자동으로 join
```

### 세마포어
```cpp
std::counting_semaphore<10> sem(3);  // 최대 카운트 10, 초기 카운트 3
sem.acquire();  // 감소
sem.release();  // 증가
```

### 배리어와 래치
```cpp
std::barrier sync_point(num_threads);
// 각 스레드:
sync_point.arrive_and_wait();  // 모든 스레드 동기화
```

### 코루틴 (C++20)
```cpp
Task<int> async_computation() {
    co_await some_async_operation();
    co_return 42;
}
```

## 권장 라이브러리

C++ 표준 라이브러리는 강력하지만, 다음 라이브러리가 도움이 될 수 있습니다:

- **Intel TBB**: 병렬 알고리즘을 위한 스레드 빌딩 블록
- **Boost.Thread**: 확장된 스레딩 유틸리티
- **Boost.Asio**: 비동기 I/O 및 네트워킹
- **folly**: 동시성 데이터 구조를 포함한 Facebook의 C++ 라이브러리
- **libcds**: Lock-Free 데이터 구조

## 도구 및 디버깅

### Thread Sanitizer
```bash
g++ -fsanitize=thread -g program.cpp
./a.out
```

### Valgrind (Helgrind)
```bash
valgrind --tool=helgrind ./program
```

### GDB 스레딩 명령어
```
info threads          # 모든 스레드 목록 표시
thread <n>            # 스레드 n으로 전환
thread apply all bt   # 모든 스레드의 백트레이스
```

## 컴파일러 지원

| 기능 | GCC | Clang | MSVC |
|---------|-----|-------|------|
| C++11 threads | 4.8+ | 3.3+ | VS2012+ |
| C++17 병렬 알고리즘 | 9+ | 완전하지 않음 | VS2017+ |
| C++20 jthread | 10+ | 14+ | VS2019 16.9+ |
| C++20 semaphore | 11+ | 11+ | VS2019 16.10+ |
| C++20 코루틴 | 10+ | 5+ | VS2019+ |

## 추가 읽기

- **C++ Concurrency in Action** by Anthony Williams (2nd edition)
- **C++ High Performance** by Björn Andrist and Viktor Sehr
- CppReference: https://en.cppreference.com/w/cpp/thread
- ISO C++ Papers: https://isocpp.org/std/status

## 탐색

- [언어 구현으로 돌아가기](../)
- 다음 주제:
  - [std::thread](./01-std-thread.md)
  - [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
  - [Atomic 연산](./03-atomic.md)
  - [Condition Variable](./04-condition-variable.md)
  - [Async와 Future](./05-async-future.md)
  - [Coroutine (C++20)](./06-coroutine.md)
