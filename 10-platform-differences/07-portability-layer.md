# 이식성 레이어 설계: 크로스 플랫폼 멀티스레딩

## 목차
1. [개요](#개요)
2. [C++11 표준 라이브러리 사용](#c11-표준-라이브러리-사용)
3. [조건부 컴파일](#조건부-컴파일)
4. [크로스 플랫폼 래퍼 설계](#크로스-플랫폼-래퍼-설계)
5. [CMake를 이용한 플랫폼 감지](#cmake를-이용한-플랫폼-감지)
6. [완전한 이식성 래퍼 예제](#완전한-이식성-래퍼-예제)
7. [서드파티 라이브러리](#서드파티-라이브러리)
8. [실용적 권장사항](#실용적-권장사항)

## 개요

크로스 플랫폼 멀티스레드 애플리케이션을 개발할 때는 플랫폼별 차이를 추상화하는 이식성 레이어가 필수적입니다. 이 문서에서는 효과적인 이식성 레이어를 설계하는 방법을 다룹니다.

### 이식성 전략

| 전략 | 장점 | 단점 | 권장 사용 |
|------|------|------|----------|
| C++11 std | 표준, 이식성 높음 | 기능 제한적 | 일반적인 경우 |
| 조건부 컴파일 | 플랫폼별 최적화 | 복잡성 증가 | 성능 중요한 경우 |
| 래퍼 라이브러리 | 깔끔한 인터페이스 | 개발 비용 | 대규모 프로젝트 |
| 서드파티 | 검증됨, 기능 풍부 | 의존성 추가 | 빠른 개발 |

## C++11 표준 라이브러리 사용

### 기본적인 std::thread 사용

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

// 스레드 함수
void worker_function(int id) {
    std::cout << "Thread " << id << " starting\n";

    // 작업 수행
    std::this_thread::sleep_for(std::chrono::seconds(1));

    std::cout << "Thread " << id << " finished\n";
}

// 클래스 멤버 함수를 스레드로 실행
class Worker {
public:
    Worker(int id) : id_(id) {}

    void run() {
        std::cout << "Worker " << id_ << " running\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }

private:
    int id_;
};

int main() {
    std::cout << "=== C++11 std::thread Example ===\n\n";

    // 1. 함수 포인터로 스레드 생성
    std::vector<std::thread> threads;

    for (int i = 0; i < 3; ++i) {
        threads.emplace_back(worker_function, i);
    }

    // 2. 람다로 스레드 생성
    threads.emplace_back([] {
        std::cout << "Lambda thread running\n";
    });

    // 3. 멤버 함수로 스레드 생성
    Worker worker(100);
    threads.emplace_back(&Worker::run, &worker);

    // 모든 스레드 대기
    for (auto& t : threads) {
        if (t.joinable()) {
            t.join();
        }
    }

    std::cout << "\nAll threads completed\n";

    return 0;
}
```

### std::mutex와 동기화

```cpp
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <chrono>

// 스레드 안전한 큐
template<typename T>
class ThreadSafeQueue {
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mutex_);
        queue_.push(std::move(value));
        cond_var_.notify_one();
    }

    bool try_pop(T& value) {
        std::lock_guard<std::mutex> lock(mutex_);
        if (queue_.empty()) {
            return false;
        }
        value = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    void wait_and_pop(T& value) {
        std::unique_lock<std::mutex> lock(mutex_);
        cond_var_.wait(lock, [this] { return !queue_.empty(); });
        value = std::move(queue_.front());
        queue_.pop();
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return queue_.empty();
    }

private:
    mutable std::mutex mutex_;
    std::queue<T> queue_;
    std::condition_variable cond_var_;
};

// 생산자-소비자 예제
void producer(ThreadSafeQueue<int>& queue, int id) {
    for (int i = 0; i < 5; ++i) {
        queue.push(id * 10 + i);
        std::cout << "Producer " << id << " pushed: " << (id * 10 + i) << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

void consumer(ThreadSafeQueue<int>& queue, int id) {
    for (int i = 0; i < 5; ++i) {
        int value;
        queue.wait_and_pop(value);
        std::cout << "Consumer " << id << " popped: " << value << "\n";
    }
}

int main() {
    std::cout << "=== Producer-Consumer with std::mutex ===\n\n";

    ThreadSafeQueue<int> queue;

    std::thread prod1(producer, std::ref(queue), 1);
    std::thread prod2(producer, std::ref(queue), 2);
    std::thread cons1(consumer, std::ref(queue), 1);
    std::thread cons2(consumer, std::ref(queue), 2);

    prod1.join();
    prod2.join();
    cons1.join();
    cons2.join();

    return 0;
}
```

### std::atomic과 락프리 프로그래밍

```cpp
#include <iostream>
#include <thread>
#include <atomic>
#include <vector>

class SpinLock {
public:
    void lock() {
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // 스핀
        }
    }

    void unlock() {
        flag_.clear(std::memory_order_release);
    }

private:
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT;
};

// 원자적 카운터
std::atomic<int> counter{0};
SpinLock spin_lock;

void increment_atomic(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

void increment_spinlock(int iterations) {
    for (int i = 0; i < iterations; ++i) {
        spin_lock.lock();
        // 임계 영역
        int temp = counter.load(std::memory_order_relaxed);
        counter.store(temp + 1, std::memory_order_relaxed);
        spin_lock.unlock();
    }
}

int main() {
    std::cout << "=== std::atomic Example ===\n\n";

    const int iterations = 100000;
    const int num_threads = 4;

    // 원자적 연산 테스트
    counter = 0;
    std::vector<std::thread> threads;

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < num_threads; ++i) {
        threads.emplace_back(increment_atomic, iterations);
    }

    for (auto& t : threads) {
        t.join();
    }

    auto end = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

    std::cout << "Atomic increment:\n";
    std::cout << "  Final counter: " << counter << " (expected: "
              << (num_threads * iterations) << ")\n";
    std::cout << "  Time: " << duration.count() << " microseconds\n";

    return 0;
}
```

### C++17/20 추가 기능

```cpp
#include <iostream>
#include <thread>
#include <shared_mutex>  // C++17
#include <vector>

// C++17: shared_mutex (read-write lock)
class SharedData {
public:
    int read() const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        return data_;
    }

    void write(int value) {
        std::unique_lock<std::shared_mutex> lock(mutex_);
        data_ = value;
    }

private:
    mutable std::shared_mutex mutex_;
    int data_ = 0;
};

// C++20: jthread (자동 join)
#if __cplusplus >= 202002L
#include <stop_token>

void interruptible_task(std::stop_token stoken) {
    while (!stoken.stop_requested()) {
        std::cout << "Working...\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    }
    std::cout << "Task interrupted\n";
}

void demonstrate_jthread() {
    std::cout << "\n=== C++20 jthread Example ===\n";

    {
        std::jthread t(interruptible_task);
        std::this_thread::sleep_for(std::chrono::seconds(2));
        // 소멸자가 자동으로 request_stop() 및 join() 호출
    }

    std::cout << "jthread automatically joined\n";
}
#endif

int main() {
    std::cout << "=== C++17 shared_mutex Example ===\n\n";

    SharedData data;

    // 읽기 스레드들 (동시에 읽을 수 있음)
    std::vector<std::thread> readers;
    for (int i = 0; i < 3; ++i) {
        readers.emplace_back([&data, i] {
            for (int j = 0; j < 5; ++j) {
                int value = data.read();
                std::cout << "Reader " << i << " read: " << value << "\n";
                std::this_thread::sleep_for(std::chrono::milliseconds(100));
            }
        });
    }

    // 쓰기 스레드
    std::thread writer([&data] {
        for (int i = 0; i < 5; ++i) {
            data.write(i);
            std::cout << "Writer wrote: " << i << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(200));
        }
    });

    for (auto& r : readers) {
        r.join();
    }
    writer.join();

    #if __cplusplus >= 202002L
    demonstrate_jthread();
    #endif

    return 0;
}
```

## 조건부 컴파일

### 플랫폼 감지 매크로

```c
// platform_detect.h
#ifndef PLATFORM_DETECT_H
#define PLATFORM_DETECT_H

// 플랫폼 감지
#if defined(_WIN32) || defined(_WIN64)
    #define PLATFORM_WINDOWS
    #ifdef _WIN64
        #define PLATFORM_WINDOWS_64
    #else
        #define PLATFORM_WINDOWS_32
    #endif
#elif defined(__APPLE__) && defined(__MACH__)
    #define PLATFORM_MACOS
    #include <TargetConditionals.h>
    #if TARGET_OS_MAC
        #define PLATFORM_OSX
    #endif
#elif defined(__linux__)
    #define PLATFORM_LINUX
    #if defined(__ANDROID__)
        #define PLATFORM_ANDROID
    #endif
#elif defined(__unix__)
    #define PLATFORM_UNIX
#elif defined(__FreeBSD__)
    #define PLATFORM_FREEBSD
#else
    #error "Unknown platform"
#endif

// POSIX 계열 플랫폼
#if defined(PLATFORM_LINUX) || defined(PLATFORM_MACOS) || \
    defined(PLATFORM_UNIX) || defined(PLATFORM_FREEBSD)
    #define PLATFORM_POSIX
#endif

// 컴파일러 감지
#if defined(_MSC_VER)
    #define COMPILER_MSVC
    #define COMPILER_VERSION _MSC_VER
#elif defined(__GNUC__)
    #define COMPILER_GCC
    #define COMPILER_VERSION (__GNUC__ * 10000 + __GNUC_MINOR__ * 100)
#elif defined(__clang__)
    #define COMPILER_CLANG
    #define COMPILER_VERSION (__clang_major__ * 10000 + __clang_minor__ * 100)
#endif

// 아키텍처 감지
#if defined(__x86_64__) || defined(_M_X64)
    #define ARCH_X64
#elif defined(__i386) || defined(_M_IX86)
    #define ARCH_X86
#elif defined(__aarch64__) || defined(_M_ARM64)
    #define ARCH_ARM64
#elif defined(__arm__) || defined(_M_ARM)
    #define ARCH_ARM
#endif

// C++ 표준 버전
#if __cplusplus >= 202002L
    #define CPP20_OR_LATER
#elif __cplusplus >= 201703L
    #define CPP17_OR_LATER
#elif __cplusplus >= 201402L
    #define CPP14_OR_LATER
#elif __cplusplus >= 201103L
    #define CPP11_OR_LATER
#endif

#endif // PLATFORM_DETECT_H
```

### 플랫폼별 헤더 포함

```c
// platform_headers.h
#ifndef PLATFORM_HEADERS_H
#define PLATFORM_HEADERS_H

#include "platform_detect.h"

// Windows 헤더
#ifdef PLATFORM_WINDOWS
    #ifndef WIN32_LEAN_AND_MEAN
        #define WIN32_LEAN_AND_MEAN
    #endif
    #include <windows.h>
    #include <process.h>
#endif

// POSIX 헤더
#ifdef PLATFORM_POSIX
    #include <pthread.h>
    #include <unistd.h>
    #include <sys/time.h>
    #include <errno.h>
#endif

// Linux 전용
#ifdef PLATFORM_LINUX
    #include <sys/syscall.h>
    #include <linux/futex.h>
#endif

// macOS 전용
#ifdef PLATFORM_MACOS
    #include <mach/mach.h>
    #include <mach/mach_time.h>
#endif

#endif // PLATFORM_HEADERS_H
```

### 조건부 함수 구현

```c
// platform_utils.h
#ifndef PLATFORM_UTILS_H
#define PLATFORM_UTILS_H

#include "platform_headers.h"

// 현재 스레드 ID 가져오기
unsigned long get_current_thread_id(void);

// 마이크로초 단위 sleep
void sleep_microseconds(unsigned long microseconds);

// 고해상도 타임스탬프
unsigned long long get_timestamp_microseconds(void);

// CPU 코어 개수
int get_cpu_count(void);

#endif // PLATFORM_UTILS_H

// platform_utils.c
#include "platform_utils.h"
#include <stdio.h>

unsigned long get_current_thread_id(void) {
    #ifdef PLATFORM_WINDOWS
        return (unsigned long)GetCurrentThreadId();
    #elif defined(PLATFORM_LINUX)
        return (unsigned long)syscall(SYS_gettid);
    #elif defined(PLATFORM_MACOS)
        uint64_t tid;
        pthread_threadid_np(NULL, &tid);
        return (unsigned long)tid;
    #else
        return (unsigned long)pthread_self();
    #endif
}

void sleep_microseconds(unsigned long microseconds) {
    #ifdef PLATFORM_WINDOWS
        Sleep(microseconds / 1000);  // Sleep은 밀리초 단위
    #else
        usleep(microseconds);
    #endif
}

unsigned long long get_timestamp_microseconds(void) {
    #ifdef PLATFORM_WINDOWS
        LARGE_INTEGER frequency, counter;
        QueryPerformanceFrequency(&frequency);
        QueryPerformanceCounter(&counter);
        return (unsigned long long)(counter.QuadPart * 1000000 / frequency.QuadPart);
    #elif defined(PLATFORM_MACOS)
        static mach_timebase_info_data_t timebase;
        if (timebase.denom == 0) {
            mach_timebase_info(&timebase);
        }
        uint64_t time = mach_absolute_time();
        return (unsigned long long)(time * timebase.numer / timebase.denom / 1000);
    #else
        struct timeval tv;
        gettimeofday(&tv, NULL);
        return (unsigned long long)(tv.tv_sec * 1000000 + tv.tv_usec);
    #endif
}

int get_cpu_count(void) {
    #ifdef PLATFORM_WINDOWS
        SYSTEM_INFO sysinfo;
        GetSystemInfo(&sysinfo);
        return (int)sysinfo.dwNumberOfProcessors;
    #elif defined(PLATFORM_POSIX)
        return (int)sysconf(_SC_NPROCESSORS_ONLN);
    #else
        return 1;  // 폴백
    #endif
}

// 사용 예제
void demonstrate_platform_utils(void) {
    printf("=== Platform Utilities ===\n\n");

    printf("Thread ID: %lu\n", get_current_thread_id());
    printf("CPU count: %d\n", get_cpu_count());

    unsigned long long start = get_timestamp_microseconds();
    sleep_microseconds(1000);  // 1ms
    unsigned long long end = get_timestamp_microseconds();

    printf("Sleep duration: %llu microseconds\n", end - start);
}
```

## 크로스 플랫폼 래퍼 설계

### 스레드 래퍼

```c
// xthread.h
#ifndef XTHREAD_H
#define XTHREAD_H

#include "platform_headers.h"

#ifdef __cplusplus
extern "C" {
#endif

// 스레드 핸들
typedef struct xthread_s* xthread_t;

// 스레드 함수 타입
#ifdef PLATFORM_WINDOWS
    typedef unsigned int (__stdcall *xthread_func_t)(void*);
#else
    typedef void* (*xthread_func_t)(void*);
#endif

// 스레드 생성
int xthread_create(xthread_t* thread, xthread_func_t func, void* arg);

// 스레드 대기
int xthread_join(xthread_t thread, void** retval);

// 스레드 분리
int xthread_detach(xthread_t thread);

// 현재 스레드
xthread_t xthread_self(void);

// 스레드 ID
unsigned long xthread_id(void);

// 스레드 우선순위
typedef enum {
    XTHREAD_PRIORITY_LOW,
    XTHREAD_PRIORITY_NORMAL,
    XTHREAD_PRIORITY_HIGH
} xthread_priority_t;

int xthread_set_priority(xthread_t thread, xthread_priority_t priority);

#ifdef __cplusplus
}
#endif

#endif // XTHREAD_H
```

```c
// xthread.c
#include "xthread.h"
#include <stdlib.h>

struct xthread_s {
    #ifdef PLATFORM_WINDOWS
        HANDLE handle;
        unsigned int thread_id;
    #else
        pthread_t handle;
    #endif
};

#ifdef PLATFORM_WINDOWS
    // Windows 래퍼 함수
    typedef struct {
        xthread_func_t func;
        void* arg;
    } thread_start_info;

    static unsigned int __stdcall thread_start(void* arg) {
        thread_start_info* info = (thread_start_info*)arg;
        xthread_func_t func = info->func;
        void* func_arg = info->arg;
        free(info);

        return func(func_arg);
    }
#endif

int xthread_create(xthread_t* thread, xthread_func_t func, void* arg) {
    xthread_t t = (xthread_t)malloc(sizeof(struct xthread_s));
    if (!t) return -1;

    #ifdef PLATFORM_WINDOWS
        thread_start_info* info = (thread_start_info*)malloc(sizeof(thread_start_info));
        if (!info) {
            free(t);
            return -1;
        }

        info->func = func;
        info->arg = arg;

        t->handle = (HANDLE)_beginthreadex(
            NULL, 0, thread_start, info, 0, &t->thread_id
        );

        if (t->handle == NULL) {
            free(info);
            free(t);
            return -1;
        }
    #else
        if (pthread_create(&t->handle, NULL, func, arg) != 0) {
            free(t);
            return -1;
        }
    #endif

    *thread = t;
    return 0;
}

int xthread_join(xthread_t thread, void** retval) {
    if (!thread) return -1;

    #ifdef PLATFORM_WINDOWS
        DWORD result = WaitForSingleObject(thread->handle, INFINITE);
        if (result != WAIT_OBJECT_0) {
            return -1;
        }

        if (retval) {
            DWORD exit_code;
            if (GetExitCodeThread(thread->handle, &exit_code)) {
                *retval = (void*)(uintptr_t)exit_code;
            }
        }

        CloseHandle(thread->handle);
    #else
        if (pthread_join(thread->handle, retval) != 0) {
            return -1;
        }
    #endif

    free(thread);
    return 0;
}

int xthread_detach(xthread_t thread) {
    if (!thread) return -1;

    #ifdef PLATFORM_WINDOWS
        CloseHandle(thread->handle);
    #else
        if (pthread_detach(thread->handle) != 0) {
            return -1;
        }
    #endif

    free(thread);
    return 0;
}

xthread_t xthread_self(void) {
    xthread_t t = (xthread_t)malloc(sizeof(struct xthread_s));
    if (!t) return NULL;

    #ifdef PLATFORM_WINDOWS
        t->handle = GetCurrentThread();
        t->thread_id = GetCurrentThreadId();
    #else
        t->handle = pthread_self();
    #endif

    return t;
}

unsigned long xthread_id(void) {
    return get_current_thread_id();
}

int xthread_set_priority(xthread_t thread, xthread_priority_t priority) {
    if (!thread) return -1;

    #ifdef PLATFORM_WINDOWS
        int win_priority;
        switch (priority) {
            case XTHREAD_PRIORITY_LOW:
                win_priority = THREAD_PRIORITY_BELOW_NORMAL;
                break;
            case XTHREAD_PRIORITY_HIGH:
                win_priority = THREAD_PRIORITY_ABOVE_NORMAL;
                break;
            default:
                win_priority = THREAD_PRIORITY_NORMAL;
        }
        return SetThreadPriority(thread->handle, win_priority) ? 0 : -1;
    #else
        // POSIX에서는 nice 값 사용
        int nice_val;
        switch (priority) {
            case XTHREAD_PRIORITY_LOW:
                nice_val = 10;
                break;
            case XTHREAD_PRIORITY_HIGH:
                nice_val = -10;
                break;
            default:
                nice_val = 0;
        }
        return setpriority(PRIO_PROCESS, 0, nice_val);
    #endif
}
```

### Mutex 래퍼

```c
// xmutex.h
#ifndef XMUTEX_H
#define XMUTEX_H

#include "platform_headers.h"

#ifdef __cplusplus
extern "C" {
#endif

typedef struct xmutex_s {
    #ifdef PLATFORM_WINDOWS
        CRITICAL_SECTION cs;
    #else
        pthread_mutex_t mutex;
    #endif
} xmutex_t;

// Mutex 초기화
int xmutex_init(xmutex_t* mutex);

// Mutex 정리
int xmutex_destroy(xmutex_t* mutex);

// Mutex 잠금
int xmutex_lock(xmutex_t* mutex);

// Mutex 잠금 시도
int xmutex_trylock(xmutex_t* mutex);

// Mutex 잠금 해제
int xmutex_unlock(xmutex_t* mutex);

#ifdef __cplusplus
}
#endif

#endif // XMUTEX_H
```

```c
// xmutex.c
#include "xmutex.h"

int xmutex_init(xmutex_t* mutex) {
    if (!mutex) return -1;

    #ifdef PLATFORM_WINDOWS
        InitializeCriticalSection(&mutex->cs);
        return 0;
    #else
        return pthread_mutex_init(&mutex->mutex, NULL);
    #endif
}

int xmutex_destroy(xmutex_t* mutex) {
    if (!mutex) return -1;

    #ifdef PLATFORM_WINDOWS
        DeleteCriticalSection(&mutex->cs);
        return 0;
    #else
        return pthread_mutex_destroy(&mutex->mutex);
    #endif
}

int xmutex_lock(xmutex_t* mutex) {
    if (!mutex) return -1;

    #ifdef PLATFORM_WINDOWS
        EnterCriticalSection(&mutex->cs);
        return 0;
    #else
        return pthread_mutex_lock(&mutex->mutex);
    #endif
}

int xmutex_trylock(xmutex_t* mutex) {
    if (!mutex) return -1;

    #ifdef PLATFORM_WINDOWS
        return TryEnterCriticalSection(&mutex->cs) ? 0 : -1;
    #else
        return pthread_mutex_trylock(&mutex->mutex);
    #endif
}

int xmutex_unlock(xmutex_t* mutex) {
    if (!mutex) return -1;

    #ifdef PLATFORM_WINDOWS
        LeaveCriticalSection(&mutex->cs);
        return 0;
    #else
        return pthread_mutex_unlock(&mutex->mutex);
    #endif
}
```

### 사용 예제

```c
// example_usage.c
#include "xthread.h"
#include "xmutex.h"
#include <stdio.h>

xmutex_t g_mutex;
int g_counter = 0;

#ifdef PLATFORM_WINDOWS
unsigned int __stdcall
#else
void*
#endif
counter_thread(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < 10000; ++i) {
        xmutex_lock(&g_mutex);
        g_counter++;
        xmutex_unlock(&g_mutex);
    }

    printf("Thread %d finished\n", id);

    #ifdef PLATFORM_WINDOWS
    return 0;
    #else
    return NULL;
    #endif
}

int main() {
    printf("=== Cross-Platform Thread Example ===\n\n");

    xmutex_init(&g_mutex);

    xthread_t threads[4];
    int ids[4] = {1, 2, 3, 4};

    for (int i = 0; i < 4; ++i) {
        if (xthread_create(&threads[i], counter_thread, &ids[i]) != 0) {
            fprintf(stderr, "Failed to create thread %d\n", i);
            return 1;
        }
    }

    for (int i = 0; i < 4; ++i) {
        xthread_join(threads[i], NULL);
    }

    printf("\nFinal counter: %d (expected: 40000)\n", g_counter);

    xmutex_destroy(&g_mutex);

    return 0;
}
```

## CMake를 이용한 플랫폼 감지

### CMakeLists.txt 예제

```cmake
cmake_minimum_required(VERSION 3.10)
project(CrossPlatformThreading C CXX)

# C/C++ 표준 설정
set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 17)

# 플랫폼 감지
if(WIN32)
    message(STATUS "Platform: Windows")
    add_definitions(-DPLATFORM_WINDOWS)
elseif(APPLE)
    message(STATUS "Platform: macOS")
    add_definitions(-DPLATFORM_MACOS)
elseif(UNIX)
    message(STATUS "Platform: Unix/Linux")
    add_definitions(-DPLATFORM_LINUX)
endif()

# 컴파일러 감지
if(MSVC)
    message(STATUS "Compiler: MSVC")
    add_compile_options(/W4)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    message(STATUS "Compiler: GCC")
    add_compile_options(-Wall -Wextra)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    message(STATUS "Compiler: Clang")
    add_compile_options(-Wall -Wextra)
endif()

# 스레드 라이브러리 찾기
find_package(Threads REQUIRED)

# 소스 파일
set(SOURCES
    platform_utils.c
    xthread.c
    xmutex.c
    example_usage.c
)

# 실행 파일 생성
add_executable(threading_example ${SOURCES})

# 스레드 라이브러리 링크
target_link_libraries(threading_example PRIVATE Threads::Threads)

# Windows 특화 설정
if(WIN32)
    target_compile_definitions(threading_example PRIVATE
        WIN32_LEAN_AND_MEAN
        _CRT_SECURE_NO_WARNINGS
    )
endif()

# POSIX 특화 설정
if(UNIX)
    target_link_libraries(threading_example PRIVATE pthread rt)
endif()

# 설치
install(TARGETS threading_example DESTINATION bin)
```

### 고급 CMake 설정

```cmake
# config.cmake.in
#ifndef CONFIG_H
#define CONFIG_H

#cmakedefine PLATFORM_WINDOWS
#cmakedefine PLATFORM_LINUX
#cmakedefine PLATFORM_MACOS

#cmakedefine HAVE_PTHREAD_H
#cmakedefine HAVE_WINDOWS_H

#define VERSION_MAJOR @PROJECT_VERSION_MAJOR@
#define VERSION_MINOR @PROJECT_VERSION_MINOR@
#define VERSION_PATCH @PROJECT_VERSION_PATCH@

#endif // CONFIG_H

# CMakeLists.txt (추가 부분)
cmake_minimum_required(VERSION 3.10)
project(CrossPlatformThreading VERSION 1.0.0)

# 기능 테스트
include(CheckIncludeFile)
check_include_file(pthread.h HAVE_PTHREAD_H)
check_include_file(windows.h HAVE_WINDOWS_H)

# 설정 파일 생성
configure_file(config.cmake.in config.h)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# 옵션
option(BUILD_TESTS "Build test suite" ON)
option(BUILD_EXAMPLES "Build examples" ON)

if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if(BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
```

## 완전한 이식성 래퍼 예제

### 통합 헤더

```c
// portable_threading.h
#ifndef PORTABLE_THREADING_H
#define PORTABLE_THREADING_H

#include "platform_detect.h"
#include "xthread.h"
#include "xmutex.h"

// 조건 변수
typedef struct xcond_s {
    #ifdef PLATFORM_WINDOWS
        CONDITION_VARIABLE cv;
    #else
        pthread_cond_t cond;
    #endif
} xcond_t;

int xcond_init(xcond_t* cond);
int xcond_destroy(xcond_t* cond);
int xcond_wait(xcond_t* cond, xmutex_t* mutex);
int xcond_signal(xcond_t* cond);
int xcond_broadcast(xcond_t* cond);

// 세마포어
typedef struct xsem_s xsem_t;

xsem_t* xsem_create(int initial_value);
void xsem_destroy(xsem_t* sem);
int xsem_wait(xsem_t* sem);
int xsem_post(xsem_t* sem);

// 원자적 연산
typedef struct {
    #ifdef PLATFORM_WINDOWS
        volatile LONG value;
    #else
        volatile int value __attribute__((aligned(4)));
    #endif
} xatomic_int_t;

void xatomic_store(xatomic_int_t* atomic, int value);
int xatomic_load(xatomic_int_t* atomic);
int xatomic_add(xatomic_int_t* atomic, int value);
int xatomic_sub(xatomic_int_t* atomic, int value);
int xatomic_compare_exchange(xatomic_int_t* atomic, int expected, int desired);

#endif // PORTABLE_THREADING_H
```

### 실전 예제: 스레드 풀

```c
// thread_pool.h
#ifndef THREAD_POOL_H
#define THREAD_POOL_H

#include "portable_threading.h"

typedef struct thread_pool_s thread_pool_t;
typedef void (*task_func_t)(void* arg);

// 스레드 풀 생성
thread_pool_t* thread_pool_create(int num_threads);

// 스레드 풀 종료
void thread_pool_destroy(thread_pool_t* pool);

// 작업 추가
int thread_pool_submit(thread_pool_t* pool, task_func_t func, void* arg);

// 대기 중인 작업 수
int thread_pool_pending_tasks(thread_pool_t* pool);

#endif // THREAD_POOL_H
```

```c
// thread_pool.c (간략화된 구현)
#include "thread_pool.h"
#include <stdlib.h>
#include <stdio.h>

typedef struct task_s {
    task_func_t func;
    void* arg;
    struct task_s* next;
} task_t;

struct thread_pool_s {
    xthread_t* threads;
    int num_threads;

    task_t* task_queue_head;
    task_t* task_queue_tail;

    xmutex_t queue_mutex;
    xcond_t queue_cond;

    int shutdown;
    xatomic_int_t pending_count;
};

#ifdef PLATFORM_WINDOWS
unsigned int __stdcall
#else
void*
#endif
worker_thread(void* arg) {
    thread_pool_t* pool = (thread_pool_t*)arg;

    while (1) {
        xmutex_lock(&pool->queue_mutex);

        while (pool->task_queue_head == NULL && !pool->shutdown) {
            xcond_wait(&pool->queue_cond, &pool->queue_mutex);
        }

        if (pool->shutdown) {
            xmutex_unlock(&pool->queue_mutex);
            break;
        }

        task_t* task = pool->task_queue_head;
        pool->task_queue_head = task->next;

        if (pool->task_queue_head == NULL) {
            pool->task_queue_tail = NULL;
        }

        xmutex_unlock(&pool->queue_mutex);

        // 작업 실행
        task->func(task->arg);
        free(task);

        xatomic_sub(&pool->pending_count, 1);
    }

    #ifdef PLATFORM_WINDOWS
    return 0;
    #else
    return NULL;
    #endif
}

thread_pool_t* thread_pool_create(int num_threads) {
    thread_pool_t* pool = (thread_pool_t*)malloc(sizeof(thread_pool_t));
    if (!pool) return NULL;

    pool->threads = (xthread_t*)malloc(sizeof(xthread_t) * num_threads);
    pool->num_threads = num_threads;
    pool->task_queue_head = NULL;
    pool->task_queue_tail = NULL;
    pool->shutdown = 0;

    xmutex_init(&pool->queue_mutex);
    xcond_init(&pool->queue_cond);
    xatomic_store(&pool->pending_count, 0);

    for (int i = 0; i < num_threads; ++i) {
        xthread_create(&pool->threads[i], worker_thread, pool);
    }

    return pool;
}

void thread_pool_destroy(thread_pool_t* pool) {
    if (!pool) return;

    xmutex_lock(&pool->queue_mutex);
    pool->shutdown = 1;
    xcond_broadcast(&pool->queue_cond);
    xmutex_unlock(&pool->queue_mutex);

    for (int i = 0; i < pool->num_threads; ++i) {
        xthread_join(pool->threads[i], NULL);
    }

    while (pool->task_queue_head) {
        task_t* task = pool->task_queue_head;
        pool->task_queue_head = task->next;
        free(task);
    }

    xmutex_destroy(&pool->queue_mutex);
    xcond_destroy(&pool->queue_cond);
    free(pool->threads);
    free(pool);
}

int thread_pool_submit(thread_pool_t* pool, task_func_t func, void* arg) {
    if (!pool || pool->shutdown) return -1;

    task_t* task = (task_t*)malloc(sizeof(task_t));
    if (!task) return -1;

    task->func = func;
    task->arg = arg;
    task->next = NULL;

    xmutex_lock(&pool->queue_mutex);

    if (pool->task_queue_tail) {
        pool->task_queue_tail->next = task;
    } else {
        pool->task_queue_head = task;
    }
    pool->task_queue_tail = task;

    xatomic_add(&pool->pending_count, 1);
    xcond_signal(&pool->queue_cond);

    xmutex_unlock(&pool->queue_mutex);

    return 0;
}

int thread_pool_pending_tasks(thread_pool_t* pool) {
    return xatomic_load(&pool->pending_count);
}
```

## 서드파티 라이브러리

### 추천 크로스 플랫폼 라이브러리

```cmake
# CMakeLists.txt에서 서드파티 라이브러리 사용

# 1. Boost.Thread
find_package(Boost REQUIRED COMPONENTS thread)
target_link_libraries(your_target PRIVATE Boost::thread)

# 2. Intel TBB
find_package(TBB REQUIRED)
target_link_libraries(your_target PRIVATE TBB::tbb)

# 3. C++11 표준 (가장 권장)
find_package(Threads REQUIRED)
target_link_libraries(your_target PRIVATE Threads::Threads)
```

## 실용적 권장사항

### 1. 가능하면 C++11 std 사용

```cpp
// 권장: 표준 라이브러리 사용
#include <thread>
#include <mutex>
#include <condition_variable>

std::mutex g_mutex;
std::thread t([]{ /* work */ });
t.join();
```

### 2. 성능이 중요한 경우만 플랫폼 API 직접 사용

```c
// 필요한 경우에만
#ifdef PLATFORM_WINDOWS
    // Windows 최적화 코드
#else
    // POSIX 코드
#endif
```

### 3. 래퍼 계층 최소화

```
Application
    |
    v
Portable API (얇은 래퍼)
    |
    v
Platform API
```

### 4. 테스트 주도 개발

```c
// 모든 플랫폼에서 동일한 테스트 수트 실행
void test_thread_creation() {
    xthread_t thread;
    assert(xthread_create(&thread, test_func, NULL) == 0);
    assert(xthread_join(thread, NULL) == 0);
}
```

## 요약

| 방법 | 권장도 | 사용 사례 |
|------|--------|----------|
| C++11 std | ⭐⭐⭐⭐⭐ | 대부분의 경우 |
| 조건부 컴파일 | ⭐⭐⭐ | 플랫폼별 최적화 필요 |
| 얇은 래퍼 | ⭐⭐⭐⭐ | C 프로젝트, 세밀한 제어 |
| 서드파티 | ⭐⭐⭐⭐ | 복잡한 기능, 빠른 개발 |

크로스 플랫폼 개발의 핵심은 **추상화와 간결함의 균형**입니다. 과도한 추상화는 복잡성을 증가시키지만, 적절한 추상화는 코드 유지보수를 크게 개선합니다.
