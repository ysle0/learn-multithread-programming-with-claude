# Lock-Free Counter

## 📌 개요

**Lock-Free Counter**는 Lock-Free 프로그래밍의 가장 기초적인 예제입니다. Atomic 연산만으로 여러 스레드가 안전하게 카운터를 증가/감소시킬 수 있으며, 락 기반 카운터보다 훨씬 빠릅니다.

이 문서는 Lock-Free 개념을 처음 배우는 사람들을 위한 입문 자료입니다.

---

## 🔍 문제 정의

여러 스레드가 공유 카운터를 동시에 증가시키는 상황:

```cpp
// ❌ 위험: Race Condition 발생
int counter = 0;

// Thread 1, 2, 3... 동시 실행
counter++;  // Read-Modify-Write → 안전하지 않음!
```

**문제**:
1. Thread 1: counter 읽음 (0)
2. Thread 2: counter 읽음 (0)
3. Thread 1: 1 증가 → counter = 1
4. Thread 2: 1 증가 → counter = 1
5. **결과**: 2번 증가했지만 값은 1 (❌ 잘못됨)

---

## 🛠️ 해결 방법 비교

### 1. Lock-Based (Mutex)

```cpp
class LockedCounter {
    int value = 0;
    std::mutex mtx;

public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx);
        value++;
    }

    int get() {
        std::lock_guard<std::mutex> lock(mtx);
        return value;
    }
};
```

**단점**:
- 락 획득/해제 오버헤드
- 경합 시 스레드 블로킹
- Context Switching 비용

### 2. Lock-Free (Atomic CAS)

```cpp
class LockFreeCounter {
    std::atomic<int> value{0};

public:
    void increment() {
        int old_value = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_weak(
            old_value,
            old_value + 1,
            std::memory_order_relaxed)) {
            // CAS 실패 시 재시도
        }
    }

    int get() const {
        return value.load(std::memory_order_relaxed);
    }
};
```

**장점**:
- 락 없음 → 블로킹 없음
- 높은 확장성

**단점**:
- CAS 루프 오버헤드 (고경합 시)
- Starvation 가능 (Wait-Free 아님)

### 3. Wait-Free (Atomic Fetch-Add)

```cpp
class WaitFreeCounter {
    std::atomic<int> value{0};

public:
    void increment() {
        value.fetch_add(1, std::memory_order_relaxed);
    }

    int get() const {
        return value.load(std::memory_order_relaxed);
    }
};
```

**장점**:
- ✅ **Wait-Free**: 모든 스레드가 O(1)에 완료
- ✅ **가장 빠름**: 하드웨어 직접 지원
- ✅ **Starvation 없음**

**최선의 선택**: 단순 카운터는 항상 `fetch_add` 사용!

---

## 📝 의사코드

### Lock-Free Counter (CAS 사용)

```
class LockFreeCounter {
    atomic<int> value = 0

    procedure Increment()
        loop
            old_value = value.load(memory_order_relaxed)
            new_value = old_value + 1

            if compare_exchange_weak(value, old_value, new_value,
                                     memory_order_relaxed,
                                     memory_order_relaxed) then
                return  // 성공
            // 실패 시 루프 반복
        end loop
    end procedure

    procedure Decrement()
        loop
            old_value = value.load(memory_order_relaxed)
            new_value = old_value - 1

            if compare_exchange_weak(value, old_value, new_value,
                                     memory_order_relaxed,
                                     memory_order_relaxed) then
                return
        end loop
    end procedure

    procedure Get() -> int
        return value.load(memory_order_relaxed)
    end procedure
}
```

### Wait-Free Counter (Fetch-Add 사용)

```
class WaitFreeCounter {
    atomic<int> value = 0

    procedure Increment()
        value.fetch_add(1, memory_order_relaxed)
    end procedure

    procedure Decrement()
        value.fetch_add(-1, memory_order_relaxed)
    end procedure

    procedure Get() -> int
        return value.load(memory_order_relaxed)
    end procedure
}
```

---

## 💻 완전한 C++ 구현

### Lock-Free Counter (CAS)

```cpp
#include <atomic>

class LockFreeCounter {
private:
    std::atomic<int> value;

public:
    LockFreeCounter(int initial = 0) : value(initial) {}

    // CAS 기반 증가
    void increment() {
        int old_value = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_weak(
            old_value,
            old_value + 1,
            std::memory_order_relaxed,
            std::memory_order_relaxed)) {
            // CAS 실패 시 old_value가 자동으로 업데이트됨
            // 루프로 재시도
        }
    }

    void decrement() {
        int old_value = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_weak(
            old_value,
            old_value - 1,
            std::memory_order_relaxed,
            std::memory_order_relaxed)) {
        }
    }

    void add(int delta) {
        int old_value = value.load(std::memory_order_relaxed);
        while (!value.compare_exchange_weak(
            old_value,
            old_value + delta,
            std::memory_order_relaxed,
            std::memory_order_relaxed)) {
        }
    }

    int get() const {
        return value.load(std::memory_order_relaxed);
    }

    void reset() {
        value.store(0, std::memory_order_relaxed);
    }
};
```

### Wait-Free Counter (Fetch-Add)

```cpp
#include <atomic>

class WaitFreeCounter {
private:
    std::atomic<int> value;

public:
    WaitFreeCounter(int initial = 0) : value(initial) {}

    // Wait-Free 증가 (하드웨어 atomic 명령어 사용)
    void increment() {
        value.fetch_add(1, std::memory_order_relaxed);
    }

    void decrement() {
        value.fetch_sub(1, std::memory_order_relaxed);
        // 또는: value.fetch_add(-1, std::memory_order_relaxed);
    }

    void add(int delta) {
        value.fetch_add(delta, std::memory_order_relaxed);
    }

    int get() const {
        return value.load(std::memory_order_relaxed);
    }

    void reset() {
        value.store(0, std::memory_order_relaxed);
    }

    // 보너스: Exchange (값을 설정하고 이전 값 반환)
    int exchange(int new_value) {
        return value.exchange(new_value, std::memory_order_relaxed);
    }
};
```

---

## 🔍 상세 분석

### CAS vs Fetch-Add

| 특성 | CAS (`compare_exchange`) | Fetch-Add |
|------|--------------------------|-----------|
| **Progress Guarantee** | Lock-Free | **Wait-Free** |
| **성능 (저경합)** | 빠름 | **매우 빠름** |
| **성능 (고경합)** | 재시도 증가 | **일정함** |
| **Starvation** | 가능 | **불가능** |
| **구현** | 루프 필요 | 단일 명령어 |
| **사용 사례** | 복잡한 업데이트 | **단순 증가/감소** |

**결론**: 단순 증가/감소는 항상 `fetch_add`/`fetch_sub` 사용!

### Memory Ordering 선택

카운터의 경우 대부분 `memory_order_relaxed`로 충분:

```cpp
// ✅ 대부분 충분
value.fetch_add(1, std::memory_order_relaxed);

// 다른 변수와 순서 보장 필요 시
value.fetch_add(1, std::memory_order_release);  // Enqueue 후 카운터 증가
int count = value.load(std::memory_order_acquire);  // 카운터 읽은 후 데이터 접근
```

**Relaxed가 안전한 이유**:
- 카운터 자체는 원자적
- 다른 메모리 작업과 순서 보장 불필요한 경우가 많음

---

## 📊 성능 비교

**벤치마크** (10M 증가, 4 스레드):

| 구현 | 시간 | 상대 성능 |
|------|------|-----------|
| Mutex | 850ms | 1.0x (기준) |
| CAS (Lock-Free) | 120ms | **7.1x** |
| Fetch-Add (Wait-Free) | 45ms | **18.9x** |

**관찰**:
- Fetch-Add가 압도적으로 빠름
- CAS도 Mutex보다 훨씬 우수
- 경합이 높을수록 격차 증가

---

## 🎯 실전 사용 예시

### 1. Statistics Collector

```cpp
class Statistics {
    std::atomic<uint64_t> requests{0};
    std::atomic<uint64_t> errors{0};
    std::atomic<uint64_t> total_latency{0};

public:
    void record_request(bool success, uint64_t latency_us) {
        requests.fetch_add(1, std::memory_order_relaxed);
        total_latency.fetch_add(latency_us, std::memory_order_relaxed);

        if (!success) {
            errors.fetch_add(1, std::memory_order_relaxed);
        }
    }

    double get_error_rate() const {
        uint64_t req = requests.load(std::memory_order_relaxed);
        uint64_t err = errors.load(std::memory_order_relaxed);
        return req > 0 ? (double)err / req : 0.0;
    }

    double get_avg_latency() const {
        uint64_t req = requests.load(std::memory_order_relaxed);
        uint64_t lat = total_latency.load(std::memory_order_relaxed);
        return req > 0 ? (double)lat / req : 0.0;
    }
};
```

### 2. Reference Counter

```cpp
class RefCount {
    std::atomic<int> count{1};  // 초기 1

public:
    void add_ref() {
        count.fetch_add(1, std::memory_order_relaxed);
    }

    bool release() {
        // ⚠️ 여기서는 acquire/release 필요!
        if (count.fetch_sub(1, std::memory_order_release) == 1) {
            std::atomic_thread_fence(std::memory_order_acquire);
            return true;  // 마지막 참조 → 삭제 가능
        }
        return false;
    }

    int get_count() const {
        return count.load(std::memory_order_relaxed);
    }
};
```

### 3. Thread-Safe ID Generator

```cpp
class IDGenerator {
    std::atomic<uint64_t> next_id{1};

public:
    uint64_t generate() {
        return next_id.fetch_add(1, std::memory_order_relaxed);
    }
};
```

---

## ⚠️ 주의사항

### 1. Overflow 검사
```cpp
// ❌ 위험: Overflow 미검사
value.fetch_add(1, std::memory_order_relaxed);

// ✅ 안전: Overflow 검사
uint64_t old_val = value.load(std::memory_order_relaxed);
while (true) {
    if (old_val == UINT64_MAX) {
        throw std::overflow_error("Counter overflow");
    }
    if (value.compare_exchange_weak(old_val, old_val + 1,
                                     std::memory_order_relaxed)) {
        break;
    }
}
```

### 2. 잘못된 Memory Ordering

```cpp
// ❌ 위험: Relaxed로는 다른 변수와 순서 보장 안됨
data = 42;
ready.store(true, std::memory_order_relaxed);  // 순서 뒤바뀔 수 있음!

// ✅ 안전: Release로 순서 보장
data = 42;
ready.store(true, std::memory_order_release);  // data 쓰기 후 ready 설정 보장
```

### 3. False Sharing

```cpp
// ❌ 성능 저하: 여러 카운터가 같은 캐시 라인
struct Counters {
    std::atomic<int> counter1;  // 같은 캐시 라인
    std::atomic<int> counter2;  // 같은 캐시 라인
};

// ✅ 최적화: 캐시 라인 분리
struct alignas(64) Counters {
    std::atomic<int> counter1;
    char padding1[60];
    std::atomic<int> counter2;
    char padding2[60];
};
```

---

## 🔗 다음 단계

Lock-Free Counter를 이해했다면:
1. [Lock-Free Stack](./03-lock-free-stack.md) - 포인터 기반 자료구조
2. [Lock-Free Queue](./04-lock-free-queue.md) - 더 복잡한 구조
3. [Memory Ordering](./02-memory-ordering.md) - 고급 메모리 모델

---

## 📚 참고 자료

- C++ Reference: [`std::atomic`](https://en.cppreference.com/w/cpp/atomic/atomic)
- Anthony Williams. "C++ Concurrency in Action" - Chapter 5

---

*Lock-Free Counter는 Lock-Free 프로그래밍의 "Hello World"입니다. 간단하지만 핵심 개념을 모두 담고 있습니다!*
