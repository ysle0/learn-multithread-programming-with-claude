# std::thread - C++ 스레딩 기초

`std::thread`는 C++에서 동시성 프로그래밍의 기본 구성 요소입니다. C++11에서 도입되었으며, OS 스레드에 대한 이식 가능한 RAII 호환 래퍼를 제공합니다.

## 목차
- [기본 개념](#기본-개념)
- [스레드 생성](#스레드-생성)
- [스레드 생명주기](#스레드-생명주기)
- [인수 전달](#인수-전달)
- [스레드 관리](#스레드-관리)
- [다른 언어와의 비교](#다른-언어와의-비교)
- [모범 사례](#모범-사례)
- [일반적인 실수](#일반적인-실수)

## 기본 개념

### std::thread란?

`std::thread`는 단일 실행 스레드를 나타냅니다. 각 `std::thread` 객체는:
- 하나의 OS 스레드에 매핑됩니다 (1:1 모델)
- 자체 스택을 가집니다 (일반적으로 ~2MB)
- 독립적으로 실행됩니다
- 소멸 전에 반드시 join 또는 detach 되어야 합니다

### 헤더와 네임스페이스
```cpp
#include <thread>
#include <iostream>

// std 네임스페이스에 있음
std::thread my_thread;
```

## 스레드 생성

### 방법 1: 함수 포인터
```cpp
#include <thread>
#include <iostream>

void hello() {
    std::cout << "Hello from thread!\n";
}

int main() {
    std::thread t(hello);
    t.join();  // 스레드 완료까지 대기
    return 0;
}
```

### 방법 2: 람다 함수
```cpp
#include <thread>
#include <iostream>

int main() {
    std::thread t([] {
        std::cout << "Hello from lambda!\n";
    });
    t.join();
    return 0;
}
```

### 방법 3: 함수 객체 (펑터)
```cpp
#include <thread>
#include <iostream>

class Worker {
public:
    void operator()() const {
        std::cout << "Hello from functor!\n";
    }
};

int main() {
    Worker w;
    std::thread t(w);  // Worker 객체 복사
    t.join();
    return 0;
}
```

### 방법 4: 멤버 함수
```cpp
#include <thread>
#include <iostream>

class Task {
public:
    void run(int n) {
        std::cout << "Task running with n=" << n << "\n";
    }
};

int main() {
    Task task;
    std::thread t(&Task::run, &task, 42);
    t.join();
    return 0;
}
```

## 스레드 생명주기

### 스레드의 상태

```cpp
#include <thread>
#include <iostream>

int main() {
    // 1. 생성됨 (아직 스레드를 나타내지 않음)
    std::thread t;
    std::cout << "Joinable: " << t.joinable() << "\n";  // false

    // 2. 실행 중
    t = std::thread([] {
        std::cout << "Working...\n";
    });
    std::cout << "Joinable: " << t.joinable() << "\n";  // true

    // 3. 조인됨 (스레드 종료, 객체가 더 이상 스레드를 나타내지 않음)
    t.join();
    std::cout << "Joinable: " << t.joinable() << "\n";  // false

    return 0;
}
```

### join vs. detach

#### join(): 스레드 완료 대기
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void work() {
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Work completed\n";
}

int main() {
    std::thread t(work);
    std::cout << "Waiting for thread...\n";
    t.join();  // 스레드가 끝날 때까지 블로킹
    std::cout << "Thread joined\n";
    return 0;
}
```

#### detach(): 실행 후 분리
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void background_work() {
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Background work done\n";
}

int main() {
    std::thread t(background_work);
    t.detach();  // 스레드가 독립적으로 계속 실행

    std::cout << "Main thread continuing...\n";
    std::this_thread::sleep_for(std::chrono::seconds(3));
    return 0;
}
// 주의: main이 먼저 종료되면 분리된 스레드가 완료되지 않을 수 있음!
```

### RAII 스레드 래퍼
```cpp
#include <thread>
#include <iostream>

class ThreadGuard {
    std::thread& t;
public:
    explicit ThreadGuard(std::thread& t_) : t(t_) {}
    ~ThreadGuard() {
        if (t.joinable()) {
            t.join();
        }
    }
    ThreadGuard(ThreadGuard const&) = delete;
    ThreadGuard& operator=(ThreadGuard const&) = delete;
};

void may_throw() {
    throw std::runtime_error("Error!");
}

int main() {
    std::thread t([] {
        std::cout << "Thread working...\n";
    });
    ThreadGuard guard(t);

    // 예외가 발생해도 guard가 스레드의 join을 보장
    may_throw();
    return 0;
}
```

### C++20 jthread: RAII 스레드
```cpp
#include <thread>
#include <iostream>

void work() {
    std::cout << "Working...\n";
}

int main() {
    std::jthread t(work);  // 소멸 시 자동으로 join
    // 명시적으로 join할 필요 없음!
    return 0;
}
```

## 인수 전달

### 값으로 전달
```cpp
#include <thread>
#include <iostream>
#include <string>

void print_string(std::string s) {
    std::cout << s << "\n";
}

int main() {
    std::string message = "Hello";
    std::thread t(print_string, message);  // message 복사
    t.join();
    return 0;
}
```

### 참조로 전달 (std::ref 사용)
```cpp
#include <thread>
#include <iostream>
#include <functional>

void increment(int& n) {
    ++n;
}

int main() {
    int value = 0;
    std::thread t(increment, std::ref(value));  // 참조로 전달
    t.join();
    std::cout << "Value: " << value << "\n";  // 1
    return 0;
}
```

### 이동 의미론
```cpp
#include <thread>
#include <iostream>
#include <memory>

void process(std::unique_ptr<int> ptr) {
    std::cout << "Processing: " << *ptr << "\n";
}

int main() {
    auto ptr = std::make_unique<int>(42);
    std::thread t(process, std::move(ptr));  // 소유권을 스레드로 이동
    // ptr은 이제 nullptr
    t.join();
    return 0;
}
```

### 여러 인수
```cpp
#include <thread>
#include <iostream>
#include <string>

void print_info(int id, const std::string& name, double value) {
    std::cout << "ID: " << id << ", Name: " << name
              << ", Value: " << value << "\n";
}

int main() {
    std::thread t(print_info, 1, "Alice", 3.14);
    t.join();
    return 0;
}
```

## 스레드 관리

### 스레드 ID 얻기
```cpp
#include <thread>
#include <iostream>

void print_thread_id() {
    std::cout << "Thread ID: " << std::this_thread::get_id() << "\n";
}

int main() {
    std::thread t1(print_thread_id);
    std::thread t2(print_thread_id);

    std::cout << "Main thread ID: " << std::this_thread::get_id() << "\n";
    std::cout << "t1 ID: " << t1.get_id() << "\n";
    std::cout << "t2 ID: " << t2.get_id() << "\n";

    t1.join();
    t2.join();
    return 0;
}
```

### 하드웨어 동시성
```cpp
#include <thread>
#include <iostream>
#include <vector>

int main() {
    unsigned int cores = std::thread::hardware_concurrency();
    std::cout << "Number of cores: " << cores << "\n";

    // 코어당 하나의 스레드 생성
    std::vector<std::thread> threads;
    for (unsigned int i = 0; i < cores; ++i) {
        threads.emplace_back([i] {
            std::cout << "Thread " << i << " on core\n";
        });
    }

    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```

### Sleep과 Yield
```cpp
#include <thread>
#include <iostream>
#include <chrono>

void busy_wait() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Busy " << i << "\n";
        std::this_thread::yield();  // 타임 슬라이스 양보
    }
}

void timed_work() {
    std::cout << "Starting...\n";
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Done after 1 second\n";

    auto wake_time = std::chrono::system_clock::now()
                   + std::chrono::milliseconds(500);
    std::this_thread::sleep_until(wake_time);
    std::cout << "Done after 500ms more\n";
}

int main() {
    std::thread t1(busy_wait);
    std::thread t2(timed_work);
    t1.join();
    t2.join();
    return 0;
}
```

### 스레드 이동
```cpp
#include <thread>
#include <iostream>

std::thread create_thread() {
    return std::thread([] {
        std::cout << "Thread from function\n";
    });
}

int main() {
    std::thread t1([] {
        std::cout << "Thread 1\n";
    });

    std::thread t2 = std::move(t1);  // t1은 더 이상 유효하지 않음
    // t1.join();  // 에러: t1은 스레드를 나타내지 않음

    std::thread t3 = create_thread();  // 반환값으로부터 이동

    t2.join();
    t3.join();
    return 0;
}
```

## 다른 언어와의 비교

### C++ vs. C#
```cpp
// C++: 명시적 join/detach 필요
std::thread t(work);
t.join();

// C# 동등 코드:
// Thread t = new Thread(Work);
// t.Start();
// t.Join();
```

### C++ vs. Go
```cpp
// C++: 무거운 OS 스레드
std::thread t(work);
t.join();

// Go: 경량 goroutine
// go work()
// (명시적 join 불필요, sync.WaitGroup 사용)
```

### C++ vs. JavaScript
```cpp
// C++: 실제 스레딩
std::thread t(work);
t.join();

// JavaScript: Worker (다른 패러다임)
// const worker = new Worker('worker.js');
// worker.postMessage('data');
```

## 모범 사례

### 1. 항상 join 또는 detach 하기
```cpp
// 좋음: 명시적 join
std::thread t(work);
t.join();

// 좋음: 명시적 detach
std::thread t(work);
t.detach();

// 좋음: jthread (C++20) 사용
std::jthread t(work);  // 자동으로 join

// 나쁨: join도 detach도 안 함
std::thread t(work);
// 소멸자가 std::terminate를 호출함!
```

### 2. 예외 안전을 위한 RAII 사용
```cpp
class ScopedThread {
    std::thread t;
public:
    explicit ScopedThread(std::thread t_) : t(std::move(t_)) {
        if (!t.joinable()) {
            throw std::logic_error("No thread");
        }
    }
    ~ScopedThread() { t.join(); }
    ScopedThread(ScopedThread const&) = delete;
};
```

### 3. 스레드 기반보다 태스크 기반 선호
```cpp
// 좋음: 태스크 기반 (더 쉽고, 에러 처리가 더 좋음)
auto future = std::async(std::launch::async, work);
future.get();

// 괜찮음: 스레드 기반 (세밀한 제어가 필요할 때)
std::thread t(work);
t.join();
```

### 4. 스레드 수 제한
```cpp
// 나쁨: 스레드가 너무 많음
for (int i = 0; i < 10000; ++i) {
    std::thread t(work);
    t.detach();
}

// 좋음: 제한된 크기의 스레드 풀
const unsigned int num_threads = std::thread::hardware_concurrency();
std::vector<std::thread> pool;
pool.reserve(num_threads);
for (unsigned int i = 0; i < num_threads; ++i) {
    pool.emplace_back(worker_function);
}
```

### 5. Thread-Local Storage 사용 시 주의
```cpp
thread_local int counter = 0;  // 각 스레드가 자체 복사본을 가짐

void increment() {
    ++counter;
    std::cout << "Thread " << std::this_thread::get_id()
              << " counter: " << counter << "\n";
}
```

## 일반적인 실수

### 1. join/detach 잊기
```cpp
// 나쁨: std::terminate를 호출함
void bad() {
    std::thread t(work);
}  // 이런!

// 좋음
void good() {
    std::jthread t(work);
}  // 자동으로 join
```

### 2. 파괴된 객체 접근
```cpp
// 나쁨: 파괴된 지역 변수에 대한 참조
void bad() {
    int value = 42;
    std::thread t([&] {
        std::cout << value << "\n";  // 정의되지 않은 동작!
    });
    t.detach();
}  // value 파괴됨, 하지만 스레드는 아직 실행 중

// 좋음: 값으로 전달하거나 수명 보장
void good() {
    int value = 42;
    std::thread t([value] {
        std::cout << value << "\n";
    });
    t.join();
}
```

### 3. 이중 join
```cpp
// 나쁨: 두 번 join할 수 없음
std::thread t(work);
t.join();
t.join();  // 정의되지 않은 동작!

// 좋음: joinable 확인
if (t.joinable()) {
    t.join();
}
```

### 4. cout에서의 경쟁 조건
```cpp
// 나쁨: 출력이 뒤섞임
std::thread t1([] {
    std::cout << "Thread 1\n";
});
std::thread t2([] {
    std::cout << "Thread 2\n";
});

// 좋음: mutex 또는 동기화 사용
std::mutex cout_mutex;
std::thread t1([&] {
    std::lock_guard lock(cout_mutex);
    std::cout << "Thread 1\n";
});
```

### 5. 스레드에서의 예외
```cpp
// 나쁨: 예외가 프로그램을 종료시킴
std::thread t([] {
    throw std::runtime_error("Error!");  // std::terminate 호출!
});
t.join();

// 좋음: 예외를 캐치하고 처리
std::thread t([] {
    try {
        throw std::runtime_error("Error!");
    } catch (const std::exception& e) {
        std::cerr << "Exception: " << e.what() << "\n";
    }
});
t.join();
```

## 내부 메커니즘

### pthread/WinAPI 래퍼 구조

`std::thread`는 플랫폼별 스레드 API의 얇은 래퍼입니다:

```cpp
// libstdc++ 내부 구조 (단순화)
class thread {
    typedef __gthread_t native_handle_type;  // pthread_t or HANDLE

    struct _State {
        virtual ~_State() = default;
        virtual void _M_run() = 0;  // 실제 작업 수행
    };

    native_handle_type _M_id;  // 스레드 핸들

public:
    template<typename _Callable, typename... _Args>
    explicit thread(_Callable&& __f, _Args&&... __args) {
        // 1. callable과 인수를 decay_copy로 저장
        // 2. __gthread_create 호출
        // 3. 실패 시 std::system_error 던짐
    }
};
```

### 스레드 생성 시스템 콜 흐름

```
std::thread 생성자
    │
    ▼
_M_start_thread() ─────────────────────────────────────────┐
    │                                                      │
    ▼ (Linux)                                              ▼ (Windows)
pthread_create()                                    CreateThread()
    │                                                      │
    ▼                                                      ▼
clone(CLONE_VM | CLONE_FS |                        NtCreateThreadEx()
      CLONE_FILES | CLONE_SIGHAND |                        │
      CLONE_THREAD | ...)                                  ▼
    │                                              커널 스레드 객체 생성
    ▼                                              스택 할당 (Reserved VM)
do_fork() → copy_process()
    │
    ▼
task_struct 할당
스택 할당 (default 8MB, guard page 포함)
TLS 영역 설정 (FS 레지스터)
```

### 스레드 스택 레이아웃 (Linux x86-64)

```
High Address
┌─────────────────────────────────────┐ ← Stack Top (pthread_attr_t.stackaddr)
│          Arguments/Env             │
├─────────────────────────────────────┤
│             Red Zone               │ ← 128 bytes (leaf function optimization)
├─────────────────────────────────────┤
│          Stack Frames              │
│    ┌─────────────────────────┐    │
│    │ Return Address         │    │
│    │ Saved RBP              │    │
│    │ Local Variables        │    │
│    │ Spilled Registers      │    │
│    └─────────────────────────┘    │
│              ...                   │
├─────────────────────────────────────┤
│         Guard Page(s)              │ ← PROT_NONE (4KB-64KB)
│   (Stack overflow detection)       │
├─────────────────────────────────────┤
│            TLS Block               │ ← FS:0 기준
│  ┌────────────────────────────┐   │
│  │ Static TLS (.tdata)       │   │ ← 음수 오프셋
│  │ pthread struct            │   │
│  │ DTV (Dynamic Thread Vector)│   │
│  └────────────────────────────┘   │
└─────────────────────────────────────┘ ← Stack Bottom
Low Address
```

### join() 내부 구현

```cpp
// pthread_join 내부 동작 (glibc)
int pthread_join(pthread_t thread, void **retval) {
    struct pthread *pd = (struct pthread *)thread;

    // 1. 이미 join 되었거나 detach 되었는지 확인
    if (pd->joinid != 0)
        return EINVAL;

    // 2. 스레드 종료 대기 (futex 기반)
    while (pd->tid != 0) {
        // FUTEX_WAIT: pd->tid가 현재 값과 같으면 sleep
        futex(&pd->tid, FUTEX_WAIT, pd->tid, NULL, NULL, 0);
    }

    // 3. 반환값 복사
    if (retval)
        *retval = pd->result;

    // 4. 리소스 정리 (스택, TLS 해제)
    __free_tcb(pd);

    return 0;
}
```

**detach()와의 차이**:
```cpp
// detach는 즉시 리소스 정리 책임을 스레드에게 넘김
int pthread_detach(pthread_t thread) {
    struct pthread *pd = (struct pthread *)thread;

    // atomic하게 joinid 설정
    // 스레드 종료 시 자체적으로 리소스 정리
    pd->joinid = pd;  // self-pointer = detached

    return 0;
}
```

### jthread Stop Token 메커니즘 (C++20)

```cpp
// std::jthread의 협력적 취소 메커니즘
class jthread {
    std::stop_source _M_stop_source;  // 취소 토큰 소스
    std::thread _M_thread;

public:
    template<typename _Callable, typename... _Args>
    explicit jthread(_Callable&& __f, _Args&&... __args) {
        // stop_token을 첫 번째 인수로 전달 (callable이 지원하면)
        if constexpr (std::is_invocable_v<_Callable, stop_token, _Args...>) {
            _M_thread = std::thread(std::forward<_Callable>(__f),
                                   _M_stop_source.get_token(),
                                   std::forward<_Args>(__args)...);
        } else {
            _M_thread = std::thread(std::forward<_Callable>(__f),
                                   std::forward<_Args>(__args)...);
        }
    }

    ~jthread() {
        if (joinable()) {
            request_stop();  // 취소 요청
            join();          // 종료 대기
        }
    }

    bool request_stop() noexcept {
        return _M_stop_source.request_stop();
    }
};

// stop_source 내부: atomic flag + callback 리스트
struct __stop_state {
    std::atomic<uint32_t> _M_owners{1};    // 참조 카운트
    std::atomic<uint32_t> _M_value{0};     // bit 0: stop requested
    __stop_callback_base* _M_callbacks{};  // 콜백 연결 리스트
    std::mutex _M_mtx;
};
```

### 하드웨어 동시성 감지

```cpp
// std::thread::hardware_concurrency() 구현
unsigned int hardware_concurrency() noexcept {
#ifdef _WIN32
    SYSTEM_INFO si;
    GetSystemInfo(&si);
    return si.dwNumberOfProcessors;
#else
    // Linux: /sys/devices/system/cpu/online 파싱 또는
    long result = sysconf(_SC_NPROCESSORS_ONLN);
    return (result > 0) ? result : 0;
#endif
}
```

**주의사항**:
- 하이퍼스레딩 시 논리 코어 수 반환 (물리 코어의 2배)
- 컨테이너/VM에서는 제한된 CPU가 아닌 호스트 CPU 수 반환할 수 있음
- NUMA 시스템에서는 노드별 CPU 친화성 고려 필요

## 성능 고려사항

### 스레드 생성 비용
- **시간**: 스레드 생성에 ~100 마이크로초
- **메모리**: 스레드당 ~2MB 스택 공간
- **시사점**: 사소한 작업에 스레드를 생성하지 마세요

### 컨텍스트 스위칭
- **비용**: 스위치당 1-10 마이크로초
- **영향**: 스레드가 많을수록 컨텍스트 스위치가 더 많아짐
- **경험 법칙**: CPU 바운드 작업에는 CPU 코어 수보다 많은 스레드를 생성하지 마세요

### 최적 스레드 수
```cpp
// CPU 바운드 작업용
unsigned int optimal = std::thread::hardware_concurrency();

// I/O 바운드 작업용 (더 많을 수 있음)
unsigned int optimal = std::thread::hardware_concurrency() * 2;
```

## 전체 예제: 병렬 합산
```cpp
#include <thread>
#include <vector>
#include <numeric>
#include <iostream>

void partial_sum(const std::vector<int>& data,
                 size_t start, size_t end,
                 long long& result) {
    result = std::accumulate(data.begin() + start,
                            data.begin() + end, 0LL);
}

int main() {
    const size_t data_size = 1'000'000;
    std::vector<int> data(data_size, 1);

    const unsigned int num_threads = std::thread::hardware_concurrency();
    std::vector<std::thread> threads;
    std::vector<long long> results(num_threads);

    size_t chunk_size = data_size / num_threads;

    // 스레드 시작
    for (unsigned int i = 0; i < num_threads; ++i) {
        size_t start = i * chunk_size;
        size_t end = (i == num_threads - 1) ? data_size
                                             : (i + 1) * chunk_size;
        threads.emplace_back(partial_sum, std::cref(data),
                           start, end, std::ref(results[i]));
    }

    // 모든 스레드 join
    for (auto& t : threads) {
        t.join();
    }

    // 결과 결합
    long long total = std::accumulate(results.begin(), results.end(), 0LL);
    std::cout << "Total sum: " << total << "\n";

    return 0;
}
```

## 추가 읽기

- [C++ Reference: std::thread](https://en.cppreference.com/w/cpp/thread/thread)
- [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
- [Async와 Future](./05-async-future.md)

## 탐색

- [C++ 개요로 돌아가기](./README.md)
- 다음: [Mutex와 Lock Guard](./02-mutex-lock-guard.md)
