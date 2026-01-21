# 동기화 기법 (Synchronization Techniques)

## 📌 개요

이 섹션에서는 멀티스레드 환경에서 **안전한 데이터 공유**를 위한 동기화 기법을 다룹니다. Race Condition을 방지하고 스레드 간 협력을 가능하게 하는 핵심 메커니즘을 학습합니다.

---

## 🎯 학습 목표

이 섹션을 완료하면 다음을 이해할 수 있습니다:

1. ✅ Mutex와 Lock을 이용한 상호 배제 (Mutual Exclusion)
2. ✅ Semaphore를 이용한 리소스 접근 제어
3. ✅ Condition Variable을 이용한 스레드 간 협력
4. ✅ Atomic Operation과 CAS를 이용한 Lock-Free 프로그래밍
5. ✅ Memory Barrier와 Memory Ordering
6. ✅ Reader-Writer Lock을 이용한 읽기/쓰기 최적화

---

## 📚 문서 목록

### [01. Mutex와 Lock](./01-mutex-lock.md)
**핵심 개념**: Mutex는 한 번에 하나의 스레드만 임계 영역(Critical Section)에 진입하도록 보장합니다.

**다루는 내용**:
- Mutex의 동작 원리
- Lock, Try-Lock, Timed-Lock
- Recursive Mutex와 일반 Mutex
- Lock Guard와 RAII 패턴
- Spinlock vs Sleeping Lock

**왜 중요한가**: 가장 기본적이고 널리 사용되는 동기화 메커니즘입니다.

---

### [02. Semaphore](./02-semaphore.md)
**핵심 개념**: Semaphore는 제한된 개수의 리소스에 대한 접근을 제어합니다.

**다루는 내용**:
- Counting Semaphore vs Binary Semaphore
- Producer-Consumer 문제
- Semaphore vs Mutex 비교
- 실전 사용 예시

**왜 중요한가**: 리소스 풀, 연결 제한 등 실전에서 자주 사용됩니다.

---

### [03. Condition Variable](./03-condition-variable.md)
**핵심 개념**: Condition Variable은 특정 조건이 만족될 때까지 스레드를 대기시키고, 조건 충족 시 깨웁니다.

**다루는 내용**:
- Wait, Notify, Broadcast
- Spurious Wakeup 문제
- Mutex와의 협력
- Producer-Consumer 패턴 구현

**왜 중요한가**: Busy-Waiting을 피하고 효율적인 스레드 협력을 구현합니다.

---

### [04. Atomic Operations](./04-atomic-operations.md)
**핵심 개념**: Atomic Operation은 중단되지 않고 완전히 실행되거나 전혀 실행되지 않는 연산입니다.

**다루는 내용**:
- Compare-And-Swap (CAS)
- Fetch-Add, Exchange
- ABA 문제
- Lock-Free 자료구조 기초

**왜 중요한가**: Lock 없이 고성능 동기화를 구현할 수 있습니다.

---

### [05. Memory Barrier](./05-memory-barrier.md)
**핵심 개념**: Memory Barrier는 메모리 연산의 순서를 보장하여 CPU 재배치를 제어합니다.

**다루는 내용**:
- Memory Reordering 문제
- Acquire-Release 시맨틱
- Sequential Consistency vs Relaxed Ordering
- Fence 명령어

**왜 중요한가**: Lock-Free 프로그래밍의 정확성을 보장합니다.

---

### [06. Reader-Writer Lock](./06-rwlock.md)
**핵심 개념**: Reader-Writer Lock은 읽기는 여러 스레드가 동시에, 쓰기는 독점적으로 수행하도록 합니다.

**다루는 내용**:
- Read Lock vs Write Lock
- Reader-Preference vs Writer-Preference
- Shared Mutex (C++17)
- 성능 최적화 전략

**왜 중요한가**: 읽기가 많은 워크로드에서 성능을 크게 향상시킵니다.

---

### [07. Futex (Fast Userspace Mutex)](./07-futex.md)
**핵심 개념**: Futex는 Linux의 저수준 동기화 프리미티브로, 유저 스페이스와 커널 스페이스를 결합한 하이브리드 메커니즘입니다.

**다루는 내용**:
- Futex의 동작 원리 (Fast Path vs Slow Path)
- Futex 기반 Mutex, Semaphore, Condition Variable 구현
- FUTEX_WAIT, FUTEX_WAKE, FUTEX_REQUEUE 연산
- Priority Inheritance (FUTEX_LOCK_PI)
- 플랫폼별 유사 메커니즘 (Windows WaitOnAddress, macOS ulock)

**왜 중요한가**: pthread mutex 등 모든 고수준 동기화 도구의 기반이며, Linux 동기화 성능의 핵심입니다.

---

## 🔍 핵심 개념 요약

### 동기화 기법 비교

| 기법 | 용도 | 성능 | 복잡도 | 사용 사례 |
|------|------|------|--------|-----------|
| **Mutex** | 상호 배제 | 중간 | 낮음 | 공유 자원 보호 |
| **Semaphore** | 리소스 카운팅 | 중간 | 중간 | 연결 풀, 리소스 제한 |
| **Condition Variable** | 조건 대기 | 높음 | 중간 | Producer-Consumer |
| **Atomic** | Lock-Free 동기화 | 매우 높음 | 높음 | 카운터, 플래그 |
| **Memory Barrier** | 순서 보장 | 높음 | 매우 높음 | Lock-Free 자료구조 |
| **RWLock** | 읽기/쓰기 분리 | 높음 (읽기 많을 때) | 중간 | 캐시, 설정 |
| **Futex** | 커널 동기화 기반 | 매우 높음 (경합 없을 때) | 매우 높음 | Mutex/Semaphore 구현 |

### 선택 가이드

```
단순 공유 자원 보호?
    └─→ Mutex + Lock Guard

제한된 리소스 관리?
    └─→ Semaphore

조건 기반 대기?
    └─→ Condition Variable + Mutex

읽기가 대부분?
    └─→ Reader-Writer Lock

최고 성능 필요? (단순 카운터, 플래그)
    └─→ Atomic Operations

Lock-Free 자료구조?
    └─→ Atomic + Memory Barrier
```

---

## 🎓 학습 경로

```
1. Mutex와 Lock (필수)
   ↓
2. Semaphore (필수)
   ↓
3. Condition Variable (필수)
   ↓
4. Atomic Operations (권장)
   ↓
5. Memory Barrier (고급)
   ↓
6. Reader-Writer Lock (권장)
   ↓
7. Futex (고급, Linux 특화)
   ↓
다음 섹션: 03-concurrency-problems/
```

**권장**: 1-3은 필수, 4-6은 성능 최적화가 필요할 때, 7은 저수준 구현을 이해하고 싶을 때 학습하세요.

---

## 💡 실전 적용 팁

### 1. 동기화 기법 선택 기준

**Mutex를 사용**:
- ✅ 간단한 공유 자원 보호
- ✅ 임계 영역이 짧음 (< 100μs)
- ✅ 대부분의 일반적인 경우

**Spinlock을 사용**:
- ✅ 임계 영역이 매우 짧음 (< 1μs)
- ✅ 컨텍스트 스위칭 비용이 더 큼
- ✅ 실시간 시스템

**Atomic을 사용**:
- ✅ 단일 변수만 업데이트
- ✅ 최고 성능 필요
- ✅ Lock-Free 알고리즘

**RWLock을 사용**:
- ✅ 읽기:쓰기 비율 > 10:1
- ✅ 임계 영역이 김 (캐시, 설정)

### 2. 일반적인 실수

❌ **Deadlock**:
```cpp
// 나쁜 예: 다른 순서로 락 획득
Thread 1: lock(A) → lock(B)
Thread 2: lock(B) → lock(A)  // Deadlock!
```

✅ **해결책**: 항상 같은 순서로 락 획득
```cpp
// 좋은 예: 일관된 순서
Thread 1: lock(A) → lock(B)
Thread 2: lock(A) → lock(B)  // OK
```

❌ **Lock 없이 공유 자원 접근**:
```cpp
int counter = 0;  // 공유 변수
void increment() {
    counter++;  // Race Condition!
}
```

✅ **해결책**:
```cpp
std::mutex mtx;
int counter = 0;
void increment() {
    std::lock_guard<std::mutex> lock(mtx);
    counter++;
}
```

### 3. 성능 최적화

**락 경합 최소화**:
- 임계 영역을 최대한 짧게 유지
- 락 밖에서 준비 작업 수행
- 락 분할 (Lock Striping)

**락 없는 알고리즘 고려**:
- 읽기 전용 데이터: 동기화 불필요
- 단일 쓰기 스레드: Atomic으로 충분
- 복잡한 경우: Lock-Free 자료구조

---

## 📊 동기화 메커니즘 상세 비용 분석

### 유저/커널 비용 구분

각 동기화 메커니즘의 비용을 **Fast Path(경합 없음)**와 **Slow Path(경합 있음)**로 구분하여 분석합니다.

| 메커니즘 | Fast Path (유저) | Slow Path (커널) | Context Switch | 특징 |
|---------|-----------------|------------------|----------------|------|
| **Atomic Load** | ~1-2 cycles (~0.5ns) | N/A | 없음 | 캐시에서 직접 읽기 |
| **Atomic CAS (성공)** | ~10-20 cycles (~5-10ns) | N/A | 없음 | LOCK prefix 사용 |
| **Atomic CAS (실패)** | ~10-20 cycles (~5-10ns) | N/A | 없음 | 재시도 필요 |
| **Memory Barrier (fence)** | ~20-40 cycles (~10-20ns) | N/A | 없음 | 파이프라인 정리 |
| **Spinlock (획득)** | ~20-50 cycles (~10-25ns) | N/A (busy-wait) | 없음 | CPU 계속 사용 |
| **Spinlock (대기)** | ~1000+ cycles/iter | N/A | 없음 | CPU 낭비 심각 |
| **Futex (fast path)** | ~25-50 cycles (~12-25ns) | 없음 | 없음 | 유저 공간 CAS만 |
| **Futex (slow path)** | ~50 cycles (유저) | ~1000-2000 cycles (커널) | 필요시 | syscall + 대기 큐 |
| **pthread_mutex (fast)** | ~50-100 cycles (~25-50ns) | 없음 | 없음 | 내부적으로 futex 사용 |
| **pthread_mutex (slow)** | ~100 cycles (유저) | ~2000-5000 cycles (커널) | 필요 | futex syscall |
| **POSIX Semaphore (fast)** | ~50-100 cycles (~25-50ns) | 없음 | 없음 | sem_wait 성공 |
| **POSIX Semaphore (slow)** | ~100 cycles (유저) | ~2000-5000 cycles (커널) | 필요 | futex 대기 |
| **Condition Variable (signal)** | ~100-200 cycles (~50-100ns) | ~1000-2000 cycles | 아님 | 대기자 깨우기 |
| **Condition Variable (wait)** | ~200 cycles (유저) | ~2000-5000 cycles (커널) | 필수 | 항상 블로킹 |
| **pthread_rwlock (read, fast)** | ~50-100 cycles (~25-50ns) | 없음 | 없음 | reader count 증가 |
| **pthread_rwlock (write, slow)** | ~100 cycles (유저) | ~2000-5000 cycles (커널) | 필요 | 모든 reader 대기 |
| **Windows CRITICAL_SECTION (fast)** | ~50-100 cycles (~25-50ns) | 없음 | 없음 | Interlocked 사용 |
| **Windows CRITICAL_SECTION (slow)** | ~100 cycles (유저) | ~3000-6000 cycles (커널) | 필요 | Event 대기 |
| **Windows Event (signal)** | ~100 cycles (유저) | ~2000-4000 cycles (커널) | 아님 | SetEvent |
| **Windows Event (wait)** | ~100 cycles (유저) | ~2000-5000 cycles (커널) | 필수 | WaitForSingleObject |
| **Context Switch** | N/A | ~3000-10000 cycles (~1.5-5μs) | - | 스케줄러 + 캐시 미스 |

### 비용 분석 요약

```
┌─────────────────────────────────────────────────────────────┐
│ 비용 계층 (낮음 → 높음)                                      │
├─────────────────────────────────────────────────────────────┤
│ 1. Atomic Load          : ~1-2 cycles    (유저)             │
│ 2. Atomic CAS           : ~10-20 cycles  (유저)             │
│ 3. Memory Barrier       : ~20-40 cycles  (유저)             │
│ 4. Spinlock (획득)      : ~20-50 cycles  (유저, 경합 없음)  │
│ 5. Futex/Mutex (fast)   : ~50-100 cycles (유저, 경합 없음)  │
│ ─────────────────────── [커널 경계] ──────────────────────── │
│ 6. Futex (slow)         : ~1000-3000 cycles (커널 호출)     │
│ 7. Mutex (slow)         : ~2000-5000 cycles (커널 블로킹)   │
│ 8. Condition Variable   : ~2000-5000 cycles (항상 커널)     │
│ 9. Context Switch       : ~3000-10000 cycles (스케줄러)     │
│ 10. Spinlock (대기)     : 무한정 증가 (CPU 낭비)            │
└─────────────────────────────────────────────────────────────┘
```

### 핵심 인사이트

1. **커널 호출은 비싸다**: 유저→커널 전환은 ~1000 cycles 이상
2. **Futex의 장점**: 경합 없을 때 커널 호출 회피 (~20배 빠름)
3. **Spinlock의 함정**: 대기 시 CPU를 계속 소비 (단기 대기만 유리)
4. **Atomic의 효율성**: 단순 연산은 lock보다 10배 이상 빠름
5. **Context Switch 비용**: 동기화 중 가장 비쌈 (~1-5μs)

---

## 🎯 동기화 메커니즘별 Use Cases

### 1. Atomic Operations

**최적 사용 사례**:
- ✅ **전역 카운터**: 요청 수, 에러 수, 통계
  ```cpp
  std::atomic<uint64_t> request_count{0};
  request_count.fetch_add(1, std::memory_order_relaxed);
  ```
- ✅ **플래그**: 초기화 완료, 종료 요청
  ```cpp
  std::atomic<bool> shutdown_requested{false};
  if (shutdown_requested.load(std::memory_order_acquire)) return;
  ```
- ✅ **단일 포인터 교체**: Lock-Free 스택 top
  ```cpp
  Node* old = top.load();
  while (!top.compare_exchange_weak(old, new_node));
  ```
- ✅ **Reference Counting**: shared_ptr 구현
  ```cpp
  ref_count.fetch_sub(1, std::memory_order_release);
  ```

**피해야 할 경우**:
- ❌ 복잡한 불변 조건 (여러 변수 업데이트)
- ❌ 긴 연산 (계산량 많은 작업)
- ❌ ABA 문제 해결 불가능한 경우

---

### 2. Spinlock

**최적 사용 사례**:
- ✅ **매우 짧은 임계 영역** (< 100 cycles, ~50ns)
  ```cpp
  spin_lock(&lock);
  shared_counter++;  // 단순 연산만
  spin_unlock(&lock);
  ```
- ✅ **인터럽트 핸들러** (커널 컨텍스트, 블로킹 불가)
  ```cpp
  // 리눅스 커널 인터럽트 핸들러
  spin_lock_irqsave(&device_lock, flags);
  device->status = READY;
  spin_unlock_irqrestore(&device_lock, flags);
  ```
- ✅ **실시간 시스템** (예측 가능한 지연 시간)
- ✅ **멀티코어 시스템** (CPU가 충분할 때)

**피해야 할 경우**:
- ❌ 긴 임계 영역 (> 1μs) → CPU 낭비
- ❌ 단일 코어 시스템 → 다른 스레드 실행 불가
- ❌ I/O 대기 → 절대 블로킹 불가
- ❌ 우선순위 역전 가능성

**성능 비교**:
```
임계 영역 50ns:  Spinlock 승리 (~70ns vs Mutex ~150ns)
임계 영역 1μs:   Mutex 승리 (~1.1μs vs Spinlock ~1.5μs)
임계 영역 10μs:  Mutex 압승 (~10.1μs vs Spinlock ~15μs+)
```

---

### 3. Mutex (pthread_mutex / Futex 기반)

**최적 사용 사례**:
- ✅ **공유 자료구조 보호**
  ```cpp
  std::mutex mutex;
  std::map<int, User> user_cache;

  void update_user(int id, const User& user) {
      std::lock_guard<std::mutex> lock(mutex);
      user_cache[id] = user;
  }
  ```
- ✅ **임계 영역이 중간 길이** (100ns ~ 100μs)
- ✅ **일반적인 동기화** (대부분의 경우)
- ✅ **블로킹이 허용되는 유저 스페이스 코드**

**실제 예시**:
```cpp
// 웹 서버 세션 관리
class SessionManager {
    std::mutex mutex_;
    std::unordered_map<std::string, Session> sessions_;

public:
    Session* get_session(const std::string& token) {
        std::lock_guard<std::mutex> lock(mutex_);  // ~50ns (fast path)
        auto it = sessions_.find(token);           // ~100ns
        return it != sessions_.end() ? &it->second : nullptr;
    }
    // 총 ~150ns - Mutex가 적합
};
```

**피해야 할 경우**:
- ❌ 매우 짧은 임계 영역 (< 50ns) → Spinlock이 더 나음
- ❌ 읽기가 대부분 → RWLock 고려
- ❌ 단일 변수만 → Atomic 사용

---

### 4. Reader-Writer Lock

**최적 사용 사례**:
- ✅ **설정 읽기** (읽기 99%, 쓰기 1%)
  ```cpp
  std::shared_mutex config_mutex;
  Config config;

  // 읽기 (여러 스레드 동시 가능)
  std::shared_lock<std::shared_mutex> lock(config_mutex);
  return config.get_value(key);
  ```
- ✅ **캐시** (읽기 빈번, 쓰기 드묾)
  ```cpp
  // 10,000 reads/sec, 10 writes/sec
  std::shared_lock lock(cache_mutex);  // 동시 읽기
  auto it = cache.find(key);
  ```
- ✅ **라우팅 테이블** (조회 빈번, 갱신 드묾)
- ✅ **읽기:쓰기 비율 > 10:1**

**성능 비교**:
```
읽기:쓰기 = 100:1
- Mutex:   평균 50ns (모든 연산 직렬화)
- RWLock:  평균 30ns (읽기 병렬화) → 40% 향상

읽기:쓰기 = 1:1
- Mutex:   평균 50ns
- RWLock:  평균 80ns (쓰기 오버헤드) → 오히려 느림
```

**피해야 할 경우**:
- ❌ 쓰기가 빈번함 (> 10%) → 일반 Mutex가 더 나음
- ❌ 임계 영역이 매우 짧음 → 오버헤드가 이득보다 큼

---

### 5. Semaphore

**최적 사용 사례**:
- ✅ **리소스 풀 관리** (DB 연결, 소켓)
  ```cpp
  sem_t connection_pool_sem;
  sem_init(&connection_pool_sem, 0, MAX_CONNECTIONS);  // 초기값 10

  // 연결 획득
  sem_wait(&connection_pool_sem);  // count--, 0이면 대기
  Connection* conn = pool.acquire();

  // 사용 후 반환
  pool.release(conn);
  sem_post(&connection_pool_sem);  // count++, 대기자 깨움
  ```
- ✅ **동시 실행 제한**
  ```cpp
  // 최대 4개 스레드만 동시 실행
  sem_t worker_limit;
  sem_init(&worker_limit, 0, 4);

  sem_wait(&worker_limit);
  perform_heavy_task();
  sem_post(&worker_limit);
  ```
- ✅ **Producer-Consumer** (카운팅 필요)
  ```cpp
  sem_t empty_slots;  // 빈 슬롯 수
  sem_t filled_slots; // 채워진 슬롯 수
  ```
- ✅ **Rate Limiting** (초당 요청 수 제한)

**피해야 할 경우**:
- ❌ 단순 상호 배제 → Mutex가 더 명확
- ❌ 조건 기반 대기 → Condition Variable 사용

---

### 6. Condition Variable

**최적 사용 사례**:
- ✅ **Producer-Consumer Queue**
  ```cpp
  std::mutex mutex;
  std::condition_variable cv;
  std::queue<Task> tasks;

  // Producer
  {
      std::lock_guard<std::mutex> lock(mutex);
      tasks.push(task);
      cv.notify_one();  // Consumer 깨우기
  }

  // Consumer
  {
      std::unique_lock<std::mutex> lock(mutex);
      cv.wait(lock, []{ return !tasks.empty(); });  // 조건 만족까지 대기
      Task t = tasks.front();
      tasks.pop();
  }
  ```
- ✅ **Thread Pool 작업 대기**
  ```cpp
  cv.wait(lock, [this]{ return stop || !tasks.empty(); });
  ```
- ✅ **이벤트 대기** (파일 준비, 데이터 도착)
- ✅ **조건 만족까지 대기** (Busy-Waiting 회피)

**vs Semaphore 비교**:
```cpp
// Semaphore: 단순 카운팅
sem_wait(&sem);  // count > 0 이면 진행

// Condition Variable: 복잡한 조건
cv.wait(lock, []{
    return queue.size() > 0 && !shutting_down;
});
```

**피해야 할 경우**:
- ❌ 단순 카운팅 → Semaphore 사용
- ❌ 짧은 폴링 → Spinlock 고려

---

### 7. Futex (저수준 프리미티브)

**최적 사용 사례**:
- ✅ **커스텀 동기화 구현** (Mutex, Semaphore 구현)
  ```c
  // 직접 Mutex 구현
  void my_mutex_lock(atomic_int* futex) {
      int c = 0;
      if ((c = atomic_exchange(futex, 1)) == 0)
          return;  // Fast path: 획득 성공

      // Slow path: 커널 대기
      do {
          if (c == 2 || atomic_exchange(futex, 2) != 0)
              futex_wait(futex, 2);
      } while ((c = atomic_exchange(futex, 2)) != 0);
  }
  ```
- ✅ **최고 성능 필요** (언어 런타임, 시스템 라이브러리)
- ✅ **플랫폼 특화 최적화**
- ✅ **glibc pthread, Go runtime, Rust std::sync**

**일반 개발자는 사용하지 말 것**:
- ❌ 복잡하고 오류 가능성 높음
- ❌ 플랫폼 종속적 (Linux 전용)
- ❌ pthread 사용으로 충분

---

### 8. Memory Barrier

**최적 사용 사례**:
- ✅ **Lock-Free 알고리즘** (순서 보장 필수)
  ```cpp
  // Double-Checked Locking
  if (instance == nullptr) {  // 1차 체크 (relaxed)
      std::lock_guard<std::mutex> lock(mutex);
      if (instance == nullptr) {
          Instance* temp = new Instance();
          std::atomic_thread_fence(std::memory_order_release);  // Barrier
          instance = temp;
      }
  }
  ```
- ✅ **Publisher-Subscriber** (메모리 순서 보장)
  ```cpp
  // Publisher
  data = new_value;
  std::atomic_thread_fence(std::memory_order_release);
  ready.store(true, std::memory_order_relaxed);

  // Subscriber
  if (ready.load(std::memory_order_relaxed)) {
      std::atomic_thread_fence(std::memory_order_acquire);
      use(data);  // data가 반드시 보임
  }
  ```
- ✅ **Store/Load 순서 강제**
- ✅ **Acquire-Release 시맨틱 구현**

**피해야 할 경우**:
- ❌ Lock 기반 코드 (Mutex가 이미 순서 보장)
- ❌ 불필요한 순서 보장 (성능 저하)

---

### 9. 플랫폼별 메커니즘

#### Windows CRITICAL_SECTION
- ✅ Windows 전용 고성능 Mutex
- ✅ pthread_mutex보다 약간 빠름 (Windows에서)
- ✅ 재귀 락 기본 지원

#### Windows Event
- ✅ Manual-Reset / Auto-Reset 이벤트
- ✅ 여러 스레드 동시 깨우기
- ✅ Condition Variable과 유사하지만 더 유연

#### eventfd (Linux)
- ✅ 파일 디스크립터 기반 이벤트
- ✅ epoll과 통합 가능
- ✅ 프로세스 간 이벤트 전달

---

## 📋 동기화 메커니즘 선택 플로우차트

```
단일 변수만 업데이트? ──YES──→ Atomic Operations
    │
    NO
    ↓
임계 영역 < 50ns? ──YES──→ Spinlock (멀티코어만)
    │
    NO
    ↓
읽기가 90% 이상? ──YES──→ Reader-Writer Lock
    │
    NO
    ↓
리소스 개수 제한? ──YES──→ Semaphore
    │
    NO
    ↓
조건 기반 대기? ──YES──→ Condition Variable + Mutex
    │
    NO
    ↓
일반적인 상호 배제 ──→ Mutex (pthread_mutex)
    │
    └──→ 최고 성능 필요? ──YES──→ 직접 Futex 구현 (전문가만)
```

---

## ⚡ 성능 벤치마크 실전 예시

### 시나리오 1: 전역 카운터 (100만 회 증가)

| 메커니즘 | 시간 | 설명 |
|---------|------|------|
| **Atomic (relaxed)** | 8 ms | 가장 빠름 |
| **Atomic (seq_cst)** | 15 ms | 순서 보장 비용 |
| **Spinlock** | 45 ms | 경합 시 CPU 낭비 |
| **Mutex** | 120 ms | 커널 호출 오버헤드 |
| **RWLock (write)** | 180 ms | Write lock 무거움 |

**결론**: 단순 카운터는 Atomic이 압도적

---

### 시나리오 2: 설정 읽기 (읽기 99%, 쓰기 1%)

| 메커니즘 | 읽기 처리량 (ops/sec) | 설명 |
|---------|---------------------|------|
| **RWLock (shared)** | 50M ops/sec | 읽기 병렬화 |
| **Mutex** | 20M ops/sec | 모든 연산 직렬화 |

**결론**: 읽기 위주는 RWLock이 2.5배 빠름

---

### 시나리오 3: Producer-Consumer (1000개 메시지)

| 메커니즘 | 지연 시간 (평균) | 설명 |
|---------|----------------|------|
| **Condition Variable** | 2.5 μs | 효율적 대기 |
| **Semaphore** | 2.8 μs | 약간 무거움 |
| **Busy-Waiting** | 50 μs | CPU 100% 낭비 |

**결론**: Condition Variable이 가장 효율적

---

## 🔑 핵심 원칙

1. **측정하라**: 추측하지 말고 프로파일링
2. **단순함 우선**: 대부분은 일반 Mutex로 충분
3. **조기 최적화 금지**: 병목이 확인된 후 최적화
4. **읽기 패턴 분석**: 읽기가 많으면 RWLock
5. **단일 변수는 Atomic**: Lock 오버헤드 회피
6. **짧은 임계 영역은 Spinlock**: 멀티코어에서만
7. **조건 대기는 CV**: Busy-Waiting 절대 금지

**교훈**: 불필요한 동기화는 성능을 크게 저하시킵니다. 올바른 도구를 선택하세요.

---

## 💻 기본 예시

### Mutex 기본 사용

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mtx;
int shared_data = 0;

void increment(int n) {
    for (int i = 0; i < n; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        shared_data++;
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Result: " << shared_data << std::endl;  // 200000
    return 0;
}
```

### Atomic 기본 사용

```cpp
#include <atomic>
#include <thread>
#include <iostream>

std::atomic<int> shared_data(0);

void increment(int n) {
    for (int i = 0; i < n; i++) {
        shared_data.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    std::thread t1(increment, 100000);
    std::thread t2(increment, 100000);

    t1.join();
    t2.join();

    std::cout << "Result: " << shared_data.load() << std::endl;  // 200000
    return 0;
}
```

### Condition Variable 기본 사용

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <queue>

std::mutex mtx;
std::condition_variable cv;
std::queue<int> queue;

void producer() {
    for (int i = 0; i < 10; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        queue.push(i);
        cv.notify_one();
    }
}

void consumer() {
    for (int i = 0; i < 10; i++) {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, []{ return !queue.empty(); });
        int value = queue.front();
        queue.pop();
        lock.unlock();
        std::cout << "Consumed: " << value << std::endl;
    }
}

int main() {
    std::thread t1(producer);
    std::thread t2(consumer);

    t1.join();
    t2.join();
    return 0;
}
```

---

## 🔗 다음 단계

동기화 기법을 이해했다면:

1. [동시성 문제](../03-concurrency-problems/README.md) - Deadlock, Livelock, Starvation
2. [동시성 패턴](../04-concurrency-patterns/README.md) - Thread Pool, Actor Model
3. [Lock-Free 프로그래밍](../05-lock-free-programming/README.md) - 고급 기법

---

## 📚 참고 자료

### 온라인
- [C++ Concurrency in Action](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition)
- [The Art of Multiprocessor Programming](https://www.elsevier.com/books/the-art-of-multiprocessor-programming/herlihy/978-0-12-415950-1)

### 표준 문서
- [C++11 Thread Support](https://en.cppreference.com/w/cpp/thread)
- [POSIX Threads](https://pubs.opengroup.org/onlinepubs/9699919799/basedefs/pthread.h.html)

---

*동기화는 멀티스레드 프로그래밍의 핵심입니다. 각 기법의 특성을 이해하고 상황에 맞게 선택하세요!*
