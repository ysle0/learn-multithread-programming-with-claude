# 디버깅 도구

## 📌 개요

멀티스레드 프로그램의 버그는 재현하기 어렵고 디버깅이 까다롭습니다. 이 문서에서는 동시성 버그를 찾고 수정하는 데 도움이 되는 도구들을 소개합니다.

---

## 🎯 동시성 버그의 특징

### 일반적인 문제들

1. **Race Condition**: 여러 스레드가 공유 데이터에 동시 접근
2. **Deadlock**: 스레드들이 서로를 기다리며 멈춤
3. **Data Race**: 동기화 없이 동일 메모리에 접근
4. **Memory Leak**: 멀티스레드 환경에서 메모리 회수 실패
5. **Livelock**: 스레드들이 서로 양보하며 진행 못함

### 디버깅의 어려움

- **간헐적 발생**: 타이밍에 의존하여 재현 어려움
- **Heisenbug**: 디버거 사용 시 사라지는 버그
- **프로덕션 전용**: 특정 환경에서만 발생

---

## 🔧 C/C++ 도구

### 1. ThreadSanitizer (TSan)

#### 개요
Google이 개발한 Data Race 검출 도구입니다.

#### 설치 및 사용

```bash
# GCC/Clang에 포함됨
# 컴파일 시 플래그 추가
g++ -fsanitize=thread -g -O1 program.cpp -o program

# 실행
./program
```

#### 예제

```cpp
#include <thread>
#include <iostream>

int shared_counter = 0;  // Race condition!

void increment() {
    for (int i = 0; i < 100000; ++i) {
        shared_counter++;  // Not atomic!
    }
}

int main() {
    std::thread t1(increment);
    std::thread t2(increment);

    t1.join();
    t2.join();

    std::cout << "Counter: " << shared_counter << std::endl;
    return 0;
}
```

**TSan 출력:**
```
==================
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 4 at 0x7b0400000010 by thread T2:
    #0 increment() program.cpp:7

  Previous write of size 4 at 0x7b0400000010 by thread T1:
    #0 increment() program.cpp:7

SUMMARY: ThreadSanitizer: data race program.cpp:7 in increment()
==================
```

#### 장점
- ✅ 매우 정확한 Data Race 검출
- ✅ 상세한 스택 트레이스
- ✅ 사용하기 쉬움

#### 단점
- ⚠️ 실행 속도 5-15배 느림
- ⚠️ 메모리 사용량 5-10배 증가
- ⚠️ Lock-Free 코드에서 False Positive 가능

---

### 2. Valgrind Helgrind

#### 개요
Data Race와 Deadlock을 검출하는 도구입니다.

#### 설치

```bash
# Ubuntu/Debian
sudo apt-get install valgrind

# macOS
brew install valgrind  # Intel Mac만 지원
```

#### 사용

```bash
# 컴파일 (디버그 심볼 포함)
g++ -g program.cpp -o program -pthread

# Helgrind로 실행
valgrind --tool=helgrind ./program

# DRD (대안 도구)
valgrind --tool=drd ./program
```

#### 예제 출력

```
==12345== Possible data race during write of size 4 at 0x601040 by thread #3
==12345== Locks held: none
==12345==    at 0x400B6E: increment() (program.cpp:7)
==12345==    by 0x4E42DC: (below main) (libc-start.c:308)
==12345==
==12345== This conflicts with a previous write of size 4 by thread #2
==12345== Locks held: none
==12345==    at 0x400B6E: increment() (program.cpp:7)
==12345==    by 0x4E42DC: (below main) (libc-start.c:308)
```

#### 장점
- ✅ Deadlock 검출 가능
- ✅ Lock 순서 문제 발견
- ✅ 설치 및 사용 간단

#### 단점
- ⚠️ 매우 느림 (20-50배)
- ⚠️ False Positive 많음
- ⚠️ 모든 플랫폼 지원 안됨 (특히 Apple Silicon)

---

### 3. AddressSanitizer (ASan)

#### 개요
메모리 오류 검출 도구 (Use-after-free, Buffer overflow 등)

#### 사용

```bash
# 컴파일
g++ -fsanitize=address -g program.cpp -o program

# 실행
./program
```

#### 멀티스레드 예제

```cpp
#include <thread>
#include <vector>

int* ptr = nullptr;

void writer() {
    ptr = new int(42);
    delete ptr;
}

void reader() {
    if (ptr) {
        int value = *ptr;  // Use-after-free!
    }
}

int main() {
    std::thread t1(writer);
    std::thread t2(reader);

    t1.join();
    t2.join();

    return 0;
}
```

**ASan 출력:**
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x6020000000d0
READ of size 4 at 0x6020000000d0 thread T2
    #0 0x400c3f in reader() program.cpp:11

0x6020000000d0 is located 0 bytes inside of 4-byte region
freed by thread T1 here:
    #0 0x7f8e9e2f3b50 in operator delete(void*)
    #1 0x400c1f in writer() program.cpp:7
```

---

### 4. GDB (GNU Debugger)

#### 멀티스레드 디버깅

```bash
# GDB 시작
gdb ./program

# 브레이크포인트 설정
(gdb) break increment
(gdb) run

# 스레드 목록 보기
(gdb) info threads

# 스레드 전환
(gdb) thread 2

# 모든 스레드의 백트레이스
(gdb) thread apply all backtrace

# 조건부 브레이크포인트
(gdb) break increment if counter > 100

# 스레드별로 명령 실행
(gdb) thread apply 1 backtrace
```

#### 멀티스레드 디버깅 팁

**1. 모든 스레드 멈추기**
```
(gdb) set scheduler-locking on
```

**2. Race Condition 재현**
```
(gdb) set scheduler-locking step
# 한 스레드씩 step
```

**3. Watch Point**
```
(gdb) watch shared_counter
# 값이 변경될 때마다 멈춤
```

---

## 🐹 Go 도구

### 1. Race Detector

#### 개요
Go 내장 Race 검출 도구입니다.

#### 사용

```bash
# 빌드 시
go build -race

# 테스트 시
go test -race

# 실행 시
go run -race main.go
```

#### 예제

```go
package main

import (
    "fmt"
    "sync"
)

var counter int  // Race!

func increment(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        counter++  // Not protected!
    }
}

func main() {
    var wg sync.WaitGroup
    wg.Add(2)

    go increment(&wg)
    go increment(&wg)

    wg.Wait()
    fmt.Println("Counter:", counter)
}
```

**출력:**
```
==================
WARNING: DATA RACE
Write at 0x00c000014098 by goroutine 7:
  main.increment()
      /path/to/main.go:12 +0x4e

Previous write at 0x00c000014098 by goroutine 6:
  main.increment()
      /path/to/main.go:12 +0x4e

Goroutine 7 (running) created at:
  main.main()
      /path/to/main.go:19 +0x8f
==================
Counter: 1543
Found 1 data race(s)
```

#### 수정 버전

```go
var (
    counter int
    mu      sync.Mutex
)

func increment(wg *sync.WaitGroup) {
    defer wg.Done()
    for i := 0; i < 1000; i++ {
        mu.Lock()
        counter++
        mu.Unlock()
    }
}

// 또는 atomic 사용
// atomic.AddInt64(&counter, 1)
```

#### 장점
- ✅ 간단한 사용법
- ✅ 정확한 검출
- ✅ Go 런타임 통합

#### 단점
- ⚠️ 실행 속도 느림
- ⚠️ 메모리 증가

---

### 2. pprof (Profiling)

#### Goroutine Profiling

```go
package main

import (
    "net/http"
    _ "net/http/pprof"
    "time"
)

func main() {
    // pprof HTTP 서버
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // 프로그램 실행
    for {
        go leakyGoroutine()
        time.Sleep(time.Second)
    }
}

func leakyGoroutine() {
    // Goroutine leak!
    select {}
}
```

**분석:**
```bash
# Goroutine 프로필
go tool pprof http://localhost:6060/debug/pprof/goroutine

# 웹 UI
go tool pprof -http=:8080 http://localhost:6060/debug/pprof/goroutine
```

#### Mutex Profiling

```go
import (
    "runtime"
)

func init() {
    // Mutex contention profiling 활성화
    runtime.SetMutexProfileFraction(1)
}

// 분석
// go tool pprof http://localhost:6060/debug/pprof/mutex
```

---

### 3. Delve (Go Debugger)

#### 설치 및 사용

```bash
# 설치
go install github.com/go-delve/delve/cmd/dlv@latest

# 디버깅 시작
dlv debug main.go

# 실행 중인 프로세스 attach
dlv attach <pid>
```

#### 명령어

```
# 브레이크포인트
(dlv) break main.increment

# Goroutine 목록
(dlv) goroutines

# 특정 Goroutine으로 전환
(dlv) goroutine 2

# 모든 Goroutine의 스택
(dlv) goroutines -t
```

---

## 🦀 Rust 도구

### 1. Miri

#### 개요
Rust의 Undefined Behavior 검출 도구입니다.

#### 설치 및 사용

```bash
# 설치
rustup +nightly component add miri

# 실행
cargo +nightly miri test
cargo +nightly miri run
```

#### 예제

```rust
use std::thread;

static mut COUNTER: i32 = 0;  // Data race!

fn main() {
    let handles: Vec<_> = (0..2)
        .map(|_| {
            thread::spawn(|| unsafe {
                COUNTER += 1;  // Unsafe!
            })
        })
        .collect();

    for h in handles {
        h.join().unwrap();
    }

    println!("Counter: {}", unsafe { COUNTER });
}
```

**Miri 출력:**
```
error: Undefined Behavior: Data race detected between Write on thread `<unnamed>` and Write on thread `<unnamed>`
```

---

## 🔍 일반 디버깅 기법

### 1. 로깅

#### 타임스탬프와 스레드 ID 포함

```cpp
#include <iostream>
#include <thread>
#include <chrono>
#include <iomanip>

void log(const std::string& msg) {
    auto now = std::chrono::system_clock::now();
    auto time_t = std::chrono::system_clock::to_time_t(now);

    std::cout << std::put_time(std::localtime(&time_t), "%H:%M:%S")
              << " [Thread " << std::this_thread::get_id() << "] "
              << msg << std::endl;
}

void worker() {
    log("Starting work");
    // ...
    log("Work complete");
}
```

#### 구조화된 로깅 (spdlog)

```cpp
#include <spdlog/spdlog.h>
#include <thread>

void worker(int id) {
    spdlog::info("Worker {} started", id);
    // ...
    spdlog::info("Worker {} finished", id);
}

int main() {
    // Thread-safe logger
    auto logger = spdlog::basic_logger_mt("multi_threaded", "logs.txt");
    spdlog::set_default_logger(logger);

    std::thread t1(worker, 1);
    std::thread t2(worker, 2);

    t1.join();
    t2.join();
}
```

---

### 2. Assertion

```cpp
#include <cassert>
#include <mutex>

std::mutex mtx;
bool locked = false;

void critical_section() {
    mtx.lock();
    locked = true;

    // 락이 획득된 상태 확인
    assert(locked && "Must hold lock!");

    // 작업...

    locked = false;
    mtx.unlock();
}
```

---

### 3. Deadlock 탐지

#### 타임아웃 사용

```cpp
#include <mutex>
#include <chrono>

std::timed_mutex mtx1, mtx2;

void safe_lock() {
    using namespace std::chrono_literals;

    if (mtx1.try_lock_for(1s)) {
        if (mtx2.try_lock_for(1s)) {
            // 작업
            mtx2.unlock();
        } else {
            std::cerr << "Failed to lock mtx2 (possible deadlock)" << std::endl;
        }
        mtx1.unlock();
    } else {
        std::cerr << "Failed to lock mtx1 (possible deadlock)" << std::endl;
    }
}
```

---

## 📊 성능 프로파일링

### 1. perf (Linux)

```bash
# 프로그램 실행 및 프로파일링
perf record -g ./program

# 결과 분석
perf report

# 특정 이벤트 측정
perf stat -e cache-misses,cache-references ./program
```

### 2. Instruments (macOS)

```bash
# Xcode Instruments 사용
# Time Profiler: CPU 사용률
# System Trace: 시스템 레벨 추적
# Leaks: 메모리 누수
```

---

## 🎓 디버깅 베스트 프랙티스

### 1. 재현 가능하게 만들기

```cpp
// 시드 고정
std::srand(42);

// 스레드 개수 고정
const int NUM_THREADS = 4;

// 실행 순서 제어 (테스트 시)
std::mutex order_mtx;
std::condition_variable order_cv;
```

### 2. 단순화하기

```cpp
// 복잡한 코드
void complex_operation() {
    lock1.lock();
    process_a();
    lock2.lock();
    process_b();
    lock2.unlock();
    lock1.unlock();
}

// 단순화된 버전
void simple_operation() {
    std::lock(lock1, lock2);  // Deadlock 방지
    std::lock_guard<std::mutex> lg1(lock1, std::adopt_lock);
    std::lock_guard<std::mutex> lg2(lock2, std::adopt_lock);

    process_a();
    process_b();
}
```

### 3. 도구 조합 사용

```bash
# TSan으로 Data Race 찾기
g++ -fsanitize=thread program.cpp && ./a.out

# ASan으로 메모리 오류 찾기
g++ -fsanitize=address program.cpp && ./a.out

# Valgrind로 Deadlock 찾기
valgrind --tool=helgrind ./a.out
```

---

## ⚠️ 주의사항

1. **프로덕션 빌드**: Sanitizer는 개발 중에만 사용
2. **성능 영향**: 디버깅 도구는 속도 저하 유발
3. **False Positive**: 일부 Lock-Free 코드는 오탐 가능
4. **플랫폼 의존**: 일부 도구는 특정 플랫폼만 지원

---

## 🔗 추가 리소스

- ThreadSanitizer: https://github.com/google/sanitizers
- Valgrind: https://valgrind.org/
- Go Race Detector: https://go.dev/doc/articles/race_detector
- Delve: https://github.com/go-delve/delve
- Miri: https://github.com/rust-lang/miri

---

*다음: [테스팅 전략](./testing-strategies.md)*
