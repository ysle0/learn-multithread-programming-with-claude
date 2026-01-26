# C++20/23 동기화 기능

## 📌 개요

C++20과 C++23은 현대적인 동시성 프로그래밍을 위한 많은 새로운 기능을 도입했습니다. 이러한 기능들은 기존 동기화 프리미티브를 보완하고, 더 안전하고 효율적인 병렬 프로그래밍을 가능하게 합니다.

**주요 추가 사항:**
- `std::jthread` - 자동 조인 스레드
- `std::stop_token` - 협력적 취소
- `std::counting_semaphore`, `std::binary_semaphore` - 세마포어
- `std::latch` - 일회용 카운트다운
- `std::barrier` - 재사용 가능한 동기화 지점
- `std::atomic<std::shared_ptr<T>>` - 원자적 스마트 포인터

## 1. std::jthread (C++20)

### 기본 개념

`std::jthread`는 "joinable thread"의 약자로, 기존 `std::thread`의 개선된 버전입니다.

**주요 장점:**
- 소멸자에서 자동 조인
- 내장 취소 메커니즘 (`std::stop_token`)
- RAII 친화적

```cpp
#include <thread>
#include <iostream>
#include <chrono>

// 기존 std::thread의 문제점
void old_way() {
    std::thread t([] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "Thread finished\n";
    });
    // 오류! join() 또는 detach() 호출 필요
    // t가 소멸되면 std::terminate() 호출됨
    t.join();  // 반드시 필요
}

// C++20: jthread 사용
void new_way() {
    std::jthread t([] {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "Thread finished\n";
    });
    // 자동으로 조인됨! (소멸자에서)
}

int main() {
    new_way();  // 안전!
    return 0;
}
```

### 협력적 취소 (Cooperative Cancellation)

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>

// stop_token을 받는 작업
void cancelable_work(std::stop_token stoken) {
    int count = 0;
    while (!stoken.stop_requested()) {
        std::cout << "Working... " << count++ << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(500));

        if (count >= 10) {
            std::cout << "Work completed naturally\n";
            break;
        }
    }

    if (stoken.stop_requested()) {
        std::cout << "Work was cancelled\n";
    }
}

int main() {
    std::jthread worker(cancelable_work);

    // 2초 후 취소 요청
    std::this_thread::sleep_for(std::chrono::seconds(2));
    worker.request_stop();

    // jthread는 자동으로 조인됨
    return 0;
}

// 출력:
// Working... 0
// Working... 1
// Working... 2
// Working... 3
// Work was cancelled
```

### stop_source와 stop_callback

```cpp
#include <thread>
#include <stop_token>
#include <iostream>
#include <chrono>
#include <vector>

class AsyncTask {
    std::jthread worker;
    std::vector<int> results;

public:
    AsyncTask() {
        worker = std::jthread([this](std::stop_token stoken) {
            // stop_callback: 취소 요청 시 자동 호출
            std::stop_callback callback(stoken, [this]() {
                std::cout << "Cleanup triggered\n";
                results.clear();
            });

            while (!stoken.stop_requested()) {
                // 작업 수행
                results.push_back(compute_something());
                std::this_thread::sleep_for(std::chrono::milliseconds(100));

                if (results.size() >= 50) {
                    break;
                }
            }

            std::cout << "Task finished with " << results.size() << " results\n";
        });
    }

    void cancel() {
        worker.request_stop();
    }

    ~AsyncTask() {
        // 자동 조인 및 정리
    }

private:
    int compute_something() {
        static int counter = 0;
        return counter++;
    }
};

int main() {
    AsyncTask task;

    std::this_thread::sleep_for(std::chrono::seconds(1));
    task.cancel();

    // task 소멸 시 자동으로 스레드 정리
    return 0;
}
```

## 2. std::counting_semaphore와 std::binary_semaphore (C++20)

### 기본 사용법

```cpp
#include <semaphore>
#include <thread>
#include <vector>
#include <iostream>

// 최대 3개의 스레드만 동시 접근 허용
std::counting_semaphore<3> pool_slots{3};

void use_resource(int id) {
    pool_slots.acquire();  // 슬롯 획득 (없으면 대기)
    {
        std::cout << "Thread " << id << " using resource\n";
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << "Thread " << id << " done\n";
    }
    pool_slots.release();  // 슬롯 반환
}

int main() {
    std::vector<std::jthread> threads;

    // 10개 스레드 생성 (하지만 최대 3개만 동시 실행)
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(use_resource, i);
    }

    return 0;  // 자동 조인
}

// 출력 (예시):
// Thread 0 using resource
// Thread 1 using resource
// Thread 2 using resource
// Thread 0 done
// Thread 3 using resource  // Thread 0의 슬롯 재사용
// ...
```

### 실전 예제: 연결 풀

```cpp
#include <semaphore>
#include <thread>
#include <queue>
#include <mutex>
#include <iostream>
#include <memory>

// 데이터베이스 연결 시뮬레이션
class DBConnection {
    int id;
public:
    explicit DBConnection(int i) : id(i) {
        std::cout << "Connection " << id << " created\n";
    }

    void query(const std::string& sql) {
        std::cout << "[Conn " << id << "] Executing: " << sql << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    ~DBConnection() {
        std::cout << "Connection " << id << " closed\n";
    }
};

class ConnectionPool {
    static constexpr int MAX_CONNECTIONS = 5;

    std::queue<std::unique_ptr<DBConnection>> available;
    std::mutex mutex;
    std::counting_semaphore<MAX_CONNECTIONS> semaphore{MAX_CONNECTIONS};
    int next_id = 0;

public:
    class Guard {
        ConnectionPool& pool;
        std::unique_ptr<DBConnection> conn;

    public:
        Guard(ConnectionPool& p, std::unique_ptr<DBConnection> c)
            : pool(p), conn(std::move(c)) {}

        DBConnection* operator->() { return conn.get(); }

        ~Guard() {
            pool.release(std::move(conn));
        }
    };

    Guard acquire() {
        semaphore.acquire();  // 연결 대기

        std::unique_lock lock(mutex);
        if (available.empty()) {
            // 새 연결 생성
            return Guard(*this, std::make_unique<DBConnection>(next_id++));
        } else {
            // 기존 연결 재사용
            auto conn = std::move(available.front());
            available.pop();
            return Guard(*this, std::move(conn));
        }
    }

private:
    void release(std::unique_ptr<DBConnection> conn) {
        {
            std::unique_lock lock(mutex);
            available.push(std::move(conn));
        }
        semaphore.release();  // 슬롯 반환
    }
};

int main() {
    ConnectionPool pool;

    std::vector<std::jthread> clients;

    // 10개 클라이언트 (하지만 최대 5개 연결만)
    for (int i = 0; i < 10; ++i) {
        clients.emplace_back([&pool, i]() {
            auto conn = pool.acquire();
            conn->query("SELECT * FROM users WHERE id = " + std::to_string(i));
        });
    }

    return 0;
}

// 출력 예시:
// Connection 0 created
// [Conn 0] Executing: SELECT * FROM users WHERE id = 0
// Connection 1 created
// [Conn 1] Executing: SELECT * FROM users WHERE id = 1
// ...
// (최대 5개 연결만 생성됨)
```

### binary_semaphore (뮤텍스 대안)

```cpp
#include <semaphore>
#include <thread>
#include <iostream>

std::binary_semaphore mutex_sem{1};  // 초기값 1
int shared_resource = 0;

void increment() {
    for (int i = 0; i < 100000; ++i) {
        mutex_sem.acquire();  // 락
        ++shared_resource;
        mutex_sem.release();  // 언락
    }
}

int main() {
    std::jthread t1(increment);
    std::jthread t2(increment);

    // 자동 조인
    std::cout << "Final value: " << shared_resource << "\n";  // 200000

    return 0;
}
```

## 3. std::latch (C++20)

### 기본 개념

`std::latch`는 일회용 카운트다운 동기화 도구입니다. 카운터가 0이 되면 대기 중인 모든 스레드가 해제됩니다.

```cpp
#include <latch>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

void worker(int id, std::latch& work_done, std::latch& start_signal) {
    std::cout << "Worker " << id << " ready\n";

    // 모든 워커가 준비될 때까지 대기
    start_signal.arrive_and_wait();

    std::cout << "Worker " << id << " working...\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(100 * id));
    std::cout << "Worker " << id << " done\n";

    // 작업 완료 신호
    work_done.count_down();
}

int main() {
    const int NUM_WORKERS = 5;

    std::latch work_done{NUM_WORKERS};      // 완료 대기용
    std::latch start_signal{NUM_WORKERS};   // 시작 신호용

    std::vector<std::jthread> workers;

    // 워커 생성
    for (int i = 0; i < NUM_WORKERS; ++i) {
        workers.emplace_back(worker, i, std::ref(work_done), std::ref(start_signal));
    }

    std::cout << "Main thread waiting for all workers to be ready...\n";

    // 모든 워커가 준비될 때까지 대기
    start_signal.wait();

    std::cout << "All workers started!\n";

    // 모든 작업 완료 대기
    work_done.wait();

    std::cout << "All work completed!\n";

    return 0;
}

// 출력:
// Worker 0 ready
// Worker 1 ready
// ...
// Main thread waiting for all workers to be ready...
// All workers started!
// Worker 0 working...
// Worker 1 working...
// ...
// All work completed!
```

### 실전 예제: 병렬 파일 처리

```cpp
#include <latch>
#include <thread>
#include <vector>
#include <iostream>
#include <fstream>
#include <filesystem>

class ParallelFileProcessor {
    std::vector<std::string> files;

public:
    explicit ParallelFileProcessor(const std::vector<std::string>& f) : files(f) {}

    void process_all() {
        std::latch completion{static_cast<std::ptrdiff_t>(files.size())};
        std::vector<std::jthread> workers;

        std::cout << "Processing " << files.size() << " files...\n";

        for (const auto& file : files) {
            workers.emplace_back([&file, &completion]() {
                process_file(file);
                completion.count_down();
            });
        }

        // 모든 파일 처리 완료 대기
        completion.wait();

        std::cout << "All files processed!\n";
    }

private:
    static void process_file(const std::string& filename) {
        std::cout << "Processing: " << filename << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        // 실제 파일 처리 로직
    }
};

int main() {
    std::vector<std::string> files = {
        "file1.txt", "file2.txt", "file3.txt",
        "file4.txt", "file5.txt"
    };

    ParallelFileProcessor processor(files);
    processor.process_all();

    return 0;
}
```

## 4. std::barrier (C++20)

### 기본 개념

`std::barrier`는 재사용 가능한 동기화 지점입니다. 모든 스레드가 도착하면 해제되고, 다음 라운드를 위해 재설정됩니다.

```cpp
#include <barrier>
#include <thread>
#include <vector>
#include <iostream>

void multi_phase_work(int id, std::barrier<>& sync_point) {
    for (int phase = 0; phase < 3; ++phase) {
        std::cout << "Thread " << id << " - Phase " << phase << " started\n";

        // 작업 수행
        std::this_thread::sleep_for(std::chrono::milliseconds(100 * (id + 1)));

        std::cout << "Thread " << id << " - Phase " << phase << " done, waiting...\n";

        // 모든 스레드가 이 지점에 도달할 때까지 대기
        sync_point.arrive_and_wait();

        std::cout << "Thread " << id << " - Phase " << phase << " synchronized!\n";
    }
}

int main() {
    const int NUM_THREADS = 3;

    // 완료 콜백 (선택적): 모든 스레드가 도착했을 때 실행
    auto on_completion = []() noexcept {
        std::cout << ">>> All threads synchronized! <<<\n";
    };

    std::barrier sync_point(NUM_THREADS, on_completion);

    std::vector<std::jthread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back(multi_phase_work, i, std::ref(sync_point));
    }

    return 0;
}

// 출력 (예시):
// Thread 0 - Phase 0 started
// Thread 1 - Phase 0 started
// Thread 2 - Phase 0 started
// Thread 0 - Phase 0 done, waiting...
// Thread 1 - Phase 0 done, waiting...
// Thread 2 - Phase 0 done, waiting...
// >>> All threads synchronized! <<<
// Thread 0 - Phase 0 synchronized!
// Thread 1 - Phase 0 synchronized!
// Thread 2 - Phase 0 synchronized!
// ...
```

### 실전 예제: 병렬 행렬 곱셈

```cpp
#include <barrier>
#include <thread>
#include <vector>
#include <iostream>

class Matrix {
    std::vector<std::vector<double>> data;
public:
    size_t rows, cols;

    Matrix(size_t r, size_t c) : rows(r), cols(c), data(r, std::vector<double>(c, 0.0)) {}

    double& at(size_t i, size_t j) { return data[i][j]; }
    const double& at(size_t i, size_t j) const { return data[i][j]; }

    void randomize() {
        for (auto& row : data) {
            for (auto& val : row) {
                val = static_cast<double>(rand() % 100);
            }
        }
    }
};

class ParallelMatrixMultiply {
    const Matrix& A;
    const Matrix& B;
    Matrix& C;
    const int num_threads;

public:
    ParallelMatrixMultiply(const Matrix& a, const Matrix& b, Matrix& c, int threads)
        : A(a), B(b), C(c), num_threads(threads) {}

    void compute() {
        std::barrier sync_point(num_threads);
        std::vector<std::jthread> workers;

        size_t rows_per_thread = A.rows / num_threads;

        for (int t = 0; t < num_threads; ++t) {
            size_t start_row = t * rows_per_thread;
            size_t end_row = (t == num_threads - 1) ? A.rows : (t + 1) * rows_per_thread;

            workers.emplace_back([this, start_row, end_row, &sync_point]() {
                // Phase 1: 행렬 곱셈 계산
                for (size_t i = start_row; i < end_row; ++i) {
                    for (size_t j = 0; j < B.cols; ++j) {
                        double sum = 0.0;
                        for (size_t k = 0; k < A.cols; ++k) {
                            sum += A.at(i, k) * B.at(k, j);
                        }
                        C.at(i, j) = sum;
                    }
                }

                // 모든 스레드가 계산 완료할 때까지 대기
                sync_point.arrive_and_wait();

                // Phase 2: 결과 검증 (선택적)
                // ...
            });
        }
    }
};

int main() {
    const size_t N = 1000;

    Matrix A(N, N), B(N, N), C(N, N);
    A.randomize();
    B.randomize();

    auto start = std::chrono::high_resolution_clock::now();

    ParallelMatrixMultiply multiply(A, B, C, std::thread::hardware_concurrency());
    multiply.compute();

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << "Matrix multiplication completed in " << duration.count() << "ms\n";

    return 0;
}
```

## 5. std::atomic<std::shared_ptr<T>> (C++20)

### 기본 사용법

```cpp
#include <atomic>
#include <memory>
#include <thread>
#include <iostream>

class Data {
    int value;
public:
    explicit Data(int v) : value(v) {
        std::cout << "Data(" << value << ") created\n";
    }
    ~Data() {
        std::cout << "Data(" << value << ") destroyed\n";
    }
    int get() const { return value; }
};

// C++20: 원자적 shared_ptr
std::atomic<std::shared_ptr<Data>> global_data;

void reader(int id) {
    for (int i = 0; i < 5; ++i) {
        // 원자적으로 로드
        auto local_data = global_data.load();

        if (local_data) {
            std::cout << "Reader " << id << " sees: " << local_data->get() << "\n";
        }

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

void writer() {
    for (int i = 0; i < 5; ++i) {
        // 새 데이터 생성
        auto new_data = std::make_shared<Data>(i);

        // 원자적으로 저장
        global_data.store(new_data);

        std::cout << "Writer updated to: " << i << "\n";

        std::this_thread::sleep_for(std::chrono::milliseconds(150));
    }
}

int main() {
    global_data.store(std::make_shared<Data>(999));

    std::jthread w(writer);
    std::jthread r1(reader, 1);
    std::jthread r2(reader, 2);

    return 0;
}
```

## 6. 성능 비교

### Semaphore vs Mutex

```cpp
#include <semaphore>
#include <mutex>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

// 성능 테스트
template<typename Sync>
auto benchmark(const std::string& name, Sync& sync, auto acquire_fn, auto release_fn) {
    const int ITERATIONS = 1'000'000;
    const int NUM_THREADS = 4;

    auto start = std::chrono::high_resolution_clock::now();

    std::vector<std::jthread> threads;
    for (int t = 0; t < NUM_THREADS; ++t) {
        threads.emplace_back([&]() {
            for (int i = 0; i < ITERATIONS / NUM_THREADS; ++i) {
                acquire_fn(sync);
                // 크리티컬 섹션 (최소)
                volatile int x = 0;
                ++x;
                release_fn(sync);
            }
        });
    }

    threads.clear();  // 자동 조인

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);

    std::cout << name << ": " << duration.count() << "ms\n";
    return duration;
}

int main() {
    std::cout << "=== 동기화 프리미티브 성능 비교 ===\n\n";

    // std::mutex
    std::mutex mtx;
    benchmark("std::mutex", mtx,
        [](auto& m) { m.lock(); },
        [](auto& m) { m.unlock(); }
    );

    // std::binary_semaphore
    std::binary_semaphore sem{1};
    benchmark("binary_semaphore", sem,
        [](auto& s) { s.acquire(); },
        [](auto& s) { s.release(); }
    );

    return 0;
}

// 출력 예시:
// === 동기화 프리미티브 성능 비교 ===
//
// std::mutex: 450ms
// binary_semaphore: 520ms
//
// mutex가 약간 더 빠름 (최적화된 구현)
```

## 7. 모범 사례

### ✅ 권장 사항

```cpp
// 1. jthread 사용 (std::thread 대신)
std::jthread worker([]() {
    // 작업
});  // 자동 조인!

// 2. stop_token으로 취소 가능하게
std::jthread worker([](std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        // 작업
    }
});

// 3. latch는 일회성, barrier는 반복용
std::latch one_time{5};       // 한 번만 사용
std::barrier<> reusable{5};   // 여러 번 재사용

// 4. 세마포어로 리소스 풀 관리
std::counting_semaphore<10> pool{10};

// 5. atomic shared_ptr로 안전한 포인터 공유
std::atomic<std::shared_ptr<Config>> config;
```

### ❌ 피해야 할 패턴

```cpp
// ❌ 1. jthread에서 join() 호출
std::jthread t(work);
t.join();  // 불필요! 자동 조인됨

// ❌ 2. latch를 재사용 시도
std::latch l{3};
l.count_down();
l.wait();
// l.count_down();  // ⚠️ 오류! 재사용 불가

// ❌ 3. barrier 없이 다단계 동기화
// barrier를 사용하면 훨씬 간단!

// ❌ 4. 세마포어 언밸런스
sem.acquire();
if (error) return;  // ⚠️ release() 누락!

// ✅ RAII 사용
class SemaphoreGuard {
    std::counting_semaphore<>& sem;
public:
    explicit SemaphoreGuard(std::counting_semaphore<>& s) : sem(s) {
        sem.acquire();
    }
    ~SemaphoreGuard() {
        sem.release();
    }
};
```

## 8. 요약

### 핵심 포인트

1. **jthread**: 자동 조인 + 취소 메커니즘
2. **stop_token**: 협력적 취소 표준화
3. **semaphore**: 리소스 카운팅 및 제한
4. **latch**: 일회용 카운트다운 동기화
5. **barrier**: 재사용 가능한 동기화 지점
6. **atomic shared_ptr**: 안전한 포인터 공유

### 빠른 참조

```cpp
// jthread - 자동 조인
std::jthread t([](std::stop_token st) {
    while (!st.stop_requested()) { /* work */ }
});
t.request_stop();

// semaphore - 리소스 제한
std::counting_semaphore<5> pool{5};
pool.acquire();
// use resource
pool.release();

// latch - 일회용 동기화
std::latch done{N};
done.count_down();  // 각 작업 완료 시
done.wait();        // 모든 작업 완료 대기

// barrier - 반복 동기화
std::barrier sync{N};
sync.arrive_and_wait();  // 각 페이즈마다

// atomic shared_ptr
std::atomic<std::shared_ptr<T>> ptr;
ptr.store(std::make_shared<T>());
auto local = ptr.load();
```

### 컴파일러 지원

| 기능 | GCC | Clang | MSVC |
|-----|-----|-------|------|
| jthread | 10+ | 14+ | VS2019 16.9+ |
| stop_token | 10+ | 14+ | VS2019 16.9+ |
| semaphore | 11+ | 11+ | VS2019 16.10+ |
| latch | 11+ | 14+ | VS2019 16.9+ |
| barrier | 11+ | 14+ | VS2019 16.10+ |
| atomic shared_ptr | 12+ | 15+ | VS2019 16.10+ |

## 참고 자료

- [C++20 Synchronization Library - cppreference](https://en.cppreference.com/w/cpp/thread)
- [P0660R10: Stop Token and Joining Thread](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0660r10.pdf)
- [P1135R6: The C++20 Synchronization Library](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1135r6.html)
- **C++20 - The Complete Guide** by Nicolai M. Josuttis
