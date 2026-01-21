# 스레드 스케줄링: Windows vs POSIX

## 목차
1. [개요](#개요)
2. [우선순위 시스템](#우선순위-시스템)
3. [스케줄링 정책](#스케줄링-정책)
4. [스레드 어피니티](#스레드-어피니티)
5. [실시간 스케줄링](#실시간-스케줄링)
6. [성능 측정과 분석](#성능-측정과-분석)
7. [실용적 권장사항](#실용적-권장사항)

## 개요

스레드 스케줄링은 운영체제가 CPU 시간을 여러 스레드에 할당하는 방식을 결정합니다. Windows와 POSIX는 서로 다른 우선순위 시스템과 스케줄링 정책을 제공합니다.

### 주요 차이점

| 항목 | Windows | POSIX |
|------|---------|-------|
| 우선순위 범위 | 0-31 (동적) | 정책에 따라 다름 |
| 프로세스 우선순위 | Priority Class (6단계) | nice 값 (-20 ~ 19) |
| 스레드 우선순위 | Relative Priority (7단계) | sched_priority |
| 스케줄링 정책 | 다단계 피드백 큐 | FIFO, RR, OTHER |
| 실시간 지원 | High/Realtime Priority | SCHED_FIFO, SCHED_RR |

## 우선순위 시스템

### Windows: Priority Class와 Thread Priority

Windows는 2단계 우선순위 시스템을 사용합니다:
1. **Priority Class**: 프로세스 레벨 (6단계)
2. **Thread Priority**: 스레드 레벨 (7단계, 상대적)

```c
#include <windows.h>
#include <stdio.h>

void PrintPriorityInfo(HANDLE hThread, const char* name) {
    int priority = GetThreadPriority(hThread);
    DWORD priorityClass = GetPriorityClass(GetCurrentProcess());

    printf("%s:\n", name);
    printf("  Priority Class: ");

    switch (priorityClass) {
        case IDLE_PRIORITY_CLASS:
            printf("IDLE (4)\n");
            break;
        case BELOW_NORMAL_PRIORITY_CLASS:
            printf("BELOW_NORMAL (6)\n");
            break;
        case NORMAL_PRIORITY_CLASS:
            printf("NORMAL (8)\n");
            break;
        case ABOVE_NORMAL_PRIORITY_CLASS:
            printf("ABOVE_NORMAL (10)\n");
            break;
        case HIGH_PRIORITY_CLASS:
            printf("HIGH (13)\n");
            break;
        case REALTIME_PRIORITY_CLASS:
            printf("REALTIME (24)\n");
            break;
    }

    printf("  Thread Priority: ");
    switch (priority) {
        case THREAD_PRIORITY_IDLE:
            printf("IDLE (base - 15)\n");
            break;
        case THREAD_PRIORITY_LOWEST:
            printf("LOWEST (base - 2)\n");
            break;
        case THREAD_PRIORITY_BELOW_NORMAL:
            printf("BELOW_NORMAL (base - 1)\n");
            break;
        case THREAD_PRIORITY_NORMAL:
            printf("NORMAL (base + 0)\n");
            break;
        case THREAD_PRIORITY_ABOVE_NORMAL:
            printf("ABOVE_NORMAL (base + 1)\n");
            break;
        case THREAD_PRIORITY_HIGHEST:
            printf("HIGHEST (base + 2)\n");
            break;
        case THREAD_PRIORITY_TIME_CRITICAL:
            printf("TIME_CRITICAL (base + 15)\n");
            break;
    }
}

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    PrintPriorityInfo(GetCurrentThread(), "Worker Thread");

    // CPU 집약적 작업
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return 0;
}

int main() {
    HANDLE hThread;

    printf("=== Initial State ===\n");
    PrintPriorityInfo(GetCurrentThread(), "Main Thread");

    // Priority Class 변경
    printf("\n=== Setting HIGH Priority Class ===\n");
    SetPriorityClass(GetCurrentProcess(), HIGH_PRIORITY_CLASS);
    PrintPriorityInfo(GetCurrentThread(), "Main Thread");

    // 스레드 생성 및 우선순위 설정
    printf("\n=== Creating Worker Thread ===\n");
    hThread = CreateThread(NULL, 0, WorkerThread, NULL, CREATE_SUSPENDED, NULL);

    SetThreadPriority(hThread, THREAD_PRIORITY_HIGHEST);
    ResumeThread(hThread);

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    // Priority Class 복원
    SetPriorityClass(GetCurrentProcess(), NORMAL_PRIORITY_CLASS);

    return 0;
}
```

### POSIX: nice와 sched_priority

POSIX는 두 가지 우선순위 메커니즘을 제공합니다:
1. **nice 값**: 프로세스/스레드 우선순위 (-20 ~ 19, 낮을수록 높은 우선순위)
2. **sched_priority**: 실시간 스케줄링 정책의 우선순위

```c
#include <pthread.h>
#include <sched.h>
#include <unistd.h>
#include <stdio.h>
#include <errno.h>
#include <sys/resource.h>

void print_priority_info(const char* name) {
    int policy;
    struct sched_param param;
    int nice_value;

    pthread_getschedparam(pthread_self(), &policy, &param);
    nice_value = getpriority(PRIO_PROCESS, 0);

    printf("%s:\n", name);
    printf("  Scheduling Policy: ");

    switch (policy) {
        case SCHED_OTHER:
            printf("SCHED_OTHER (normal)\n");
            break;
        case SCHED_FIFO:
            printf("SCHED_FIFO (realtime)\n");
            break;
        case SCHED_RR:
            printf("SCHED_RR (realtime)\n");
            break;
        #ifdef SCHED_BATCH
        case SCHED_BATCH:
            printf("SCHED_BATCH (batch)\n");
            break;
        #endif
        #ifdef SCHED_IDLE
        case SCHED_IDLE:
            printf("SCHED_IDLE (idle)\n");
            break;
        #endif
    }

    printf("  sched_priority: %d\n", param.sched_priority);
    printf("  nice value: %d\n", nice_value);
}

void* worker_thread(void* arg) {
    print_priority_info("Worker Thread");

    // CPU 집약적 작업
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return NULL;
}

int main() {
    pthread_t thread;
    pthread_attr_t attr;
    struct sched_param param;
    int policy;

    printf("=== Initial State ===\n");
    print_priority_info("Main Thread");

    // nice 값 변경 (root 권한 필요할 수 있음)
    printf("\n=== Setting nice value ===\n");
    if (setpriority(PRIO_PROCESS, 0, -10) == -1) {
        printf("Failed to set nice value (may need root): %s\n",
               strerror(errno));
    } else {
        print_priority_info("Main Thread");
    }

    // 스레드 속성 초기화
    pthread_attr_init(&attr);

    // 스케줄링 정책과 우선순위 설정
    printf("\n=== Creating Worker Thread ===\n");

    // 정책 설정 (SCHED_FIFO는 root 권한 필요)
    policy = SCHED_OTHER;
    pthread_attr_setschedpolicy(&attr, policy);

    // 우선순위 설정
    param.sched_priority = sched_get_priority_max(policy);
    pthread_attr_setschedparam(&attr, &param);

    // 상속 정책 설정
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

    // 스레드 생성
    if (pthread_create(&thread, &attr, worker_thread, NULL) != 0) {
        printf("Failed to create thread with priority (may need root)\n");
        // 기본 속성으로 재시도
        pthread_create(&thread, NULL, worker_thread, NULL);
    }

    pthread_join(thread, NULL);
    pthread_attr_destroy(&attr);

    return 0;
}
```

### 우선순위 매핑 비교

| Windows Priority | 실제 값 | POSIX 상응 | 설명 |
|-----------------|---------|-----------|------|
| IDLE | 1 | nice 19, SCHED_IDLE | 최저 우선순위 |
| LOWEST | base-2 | nice 10-19 | 매우 낮음 |
| BELOW_NORMAL | base-1 | nice 5-9 | 낮음 |
| NORMAL | base | nice 0 | 기본값 |
| ABOVE_NORMAL | base+1 | nice -1 ~ -4 | 높음 |
| HIGHEST | base+2 | nice -5 ~ -10 | 매우 높음 |
| TIME_CRITICAL | base+15 | SCHED_FIFO/RR | 실시간 |

## 스케줄링 정책

### Windows: Multilevel Feedback Queue

Windows는 다단계 피드백 큐를 사용하며, 우선순위 부스팅을 동적으로 수행합니다.

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI CPUBoundThread(LPVOID lpParam) {
    int id = *(int*)lpParam;
    DWORD startTime = GetTickCount();

    volatile long sum = 0;
    for (long i = 0; i < 500000000; i++) {
        sum += i;
    }

    DWORD endTime = GetTickCount();
    printf("CPU-bound thread %d: %lu ms\n", id, endTime - startTime);

    return 0;
}

DWORD WINAPI IOBoundThread(LPVOID lpParam) {
    int id = *(int*)lpParam;
    DWORD startTime = GetTickCount();

    // I/O 시뮬레이션 (Sleep)
    for (int i = 0; i < 100; i++) {
        Sleep(1);
        // 약간의 작업
        volatile long sum = 0;
        for (long j = 0; j < 1000000; j++) {
            sum += j;
        }
    }

    DWORD endTime = GetTickCount();
    printf("I/O-bound thread %d: %lu ms\n", id, endTime - startTime);

    return 0;
}

int main() {
    HANDLE threads[4];
    int ids[4] = {1, 2, 3, 4};

    printf("=== Testing Priority Boost ===\n");
    printf("Windows automatically boosts priority of I/O-bound threads\n\n");

    // 2개의 CPU-bound 스레드
    threads[0] = CreateThread(NULL, 0, CPUBoundThread, &ids[0], 0, NULL);
    threads[1] = CreateThread(NULL, 0, CPUBoundThread, &ids[1], 0, NULL);

    // 2개의 I/O-bound 스레드
    threads[2] = CreateThread(NULL, 0, IOBoundThread, &ids[2], 0, NULL);
    threads[3] = CreateThread(NULL, 0, IOBoundThread, &ids[3], 0, NULL);

    WaitForMultipleObjects(4, threads, TRUE, INFINITE);

    for (int i = 0; i < 4; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

### POSIX: 스케줄링 정책 선택

```c
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>
#include <string.h>

void* cpu_bound_thread(void* arg) {
    int id = *(int*)arg;
    struct timeval start, end;

    gettimeofday(&start, NULL);

    volatile long sum = 0;
    for (long i = 0; i < 500000000; i++) {
        sum += i;
    }

    gettimeofday(&end, NULL);
    long elapsed = (end.tv_sec - start.tv_sec) * 1000 +
                   (end.tv_usec - start.tv_usec) / 1000;

    printf("CPU-bound thread %d: %ld ms\n", id, elapsed);

    return NULL;
}

void* io_bound_thread(void* arg) {
    int id = *(int*)arg;
    struct timeval start, end;

    gettimeofday(&start, NULL);

    // I/O 시뮬레이션
    for (int i = 0; i < 100; i++) {
        usleep(1000);
        // 약간의 작업
        volatile long sum = 0;
        for (long j = 0; j < 1000000; j++) {
            sum += j;
        }
    }

    gettimeofday(&end, NULL);
    long elapsed = (end.tv_sec - start.tv_sec) * 1000 +
                   (end.tv_usec - start.tv_usec) / 1000;

    printf("I/O-bound thread %d: %ld ms\n", id, elapsed);

    return NULL;
}

void create_thread_with_policy(pthread_t* thread, void* (*func)(void*),
                               int* id, int policy, int priority) {
    pthread_attr_t attr;
    struct sched_param param;

    pthread_attr_init(&attr);
    pthread_attr_setschedpolicy(&attr, policy);

    param.sched_priority = priority;
    pthread_attr_setschedparam(&attr, &param);
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

    if (pthread_create(thread, &attr, func, id) != 0) {
        // Root 권한이 없으면 기본 속성으로
        pthread_create(thread, NULL, func, id);
    }

    pthread_attr_destroy(&attr);
}

int main() {
    pthread_t threads[4];
    int ids[4] = {1, 2, 3, 4};

    printf("=== Testing POSIX Scheduling Policies ===\n");
    printf("Note: SCHED_FIFO/RR require root privileges\n\n");

    int policy = SCHED_OTHER;
    int min_prio = sched_get_priority_min(policy);
    int max_prio = sched_get_priority_max(policy);

    printf("Policy: SCHED_OTHER, Priority range: %d - %d\n\n",
           min_prio, max_prio);

    // 스레드 생성
    create_thread_with_policy(&threads[0], cpu_bound_thread,
                             &ids[0], policy, min_prio);
    create_thread_with_policy(&threads[1], cpu_bound_thread,
                             &ids[1], policy, min_prio);
    create_thread_with_policy(&threads[2], io_bound_thread,
                             &ids[2], policy, min_prio);
    create_thread_with_policy(&threads[3], io_bound_thread,
                             &ids[3], policy, min_prio);

    // 대기
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### POSIX 스케줄링 정책 비교

```c
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/time.h>

void* test_thread(void* arg) {
    const char* name = (const char*)arg;
    int policy;
    struct sched_param param;

    pthread_getschedparam(pthread_self(), &policy, &param);

    printf("%s:\n", name);
    printf("  Policy: %s\n",
           policy == SCHED_OTHER ? "SCHED_OTHER" :
           policy == SCHED_FIFO ? "SCHED_FIFO" :
           policy == SCHED_RR ? "SCHED_RR" : "UNKNOWN");
    printf("  Priority: %d\n", param.sched_priority);

    // 간단한 작업
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return NULL;
}

void test_policy(const char* name, int policy) {
    pthread_t thread;
    pthread_attr_t attr;
    struct sched_param param;

    printf("\n=== Testing %s ===\n", name);

    pthread_attr_init(&attr);
    pthread_attr_setschedpolicy(&attr, policy);
    pthread_attr_setinheritsched(&attr, PTHREAD_EXPLICIT_SCHED);

    // 중간 우선순위 설정
    int min = sched_get_priority_min(policy);
    int max = sched_get_priority_max(policy);
    param.sched_priority = (min + max) / 2;

    pthread_attr_setschedparam(&attr, &param);

    printf("Priority range: %d - %d\n", min, max);

    if (pthread_create(&thread, &attr, test_thread, (void*)name) != 0) {
        printf("Failed to create thread (may need root privileges)\n");
    } else {
        pthread_join(thread, NULL);
    }

    pthread_attr_destroy(&attr);
}

int main() {
    printf("POSIX Scheduling Policies Comparison\n");

    test_policy("SCHED_OTHER", SCHED_OTHER);
    test_policy("SCHED_FIFO", SCHED_FIFO);
    test_policy("SCHED_RR", SCHED_RR);

    #ifdef SCHED_BATCH
    test_policy("SCHED_BATCH", SCHED_BATCH);
    #endif

    #ifdef SCHED_IDLE
    test_policy("SCHED_IDLE", SCHED_IDLE);
    #endif

    return 0;
}
```

## 스레드 어피니티

### Windows: Processor Affinity

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI AffinityThread(LPVOID lpParam) {
    int id = *(int*)lpParam;
    DWORD_PTR affinityMask;

    // 현재 어피니티 가져오기
    HANDLE hThread = GetCurrentThread();
    DWORD_PTR processAffinityMask, systemAffinityMask;

    GetProcessAffinityMask(GetCurrentProcess(),
                          &processAffinityMask,
                          &systemAffinityMask);

    printf("Thread %d:\n", id);
    printf("  Process affinity mask: 0x%llx\n", processAffinityMask);
    printf("  System affinity mask: 0x%llx\n", systemAffinityMask);

    // 이전 어피니티 가져오기
    affinityMask = SetThreadAffinityMask(hThread, processAffinityMask);
    printf("  Previous thread affinity: 0x%llx\n", affinityMask);

    // 작업 수행
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return 0;
}

int main() {
    HANDLE threads[4];
    int ids[4] = {0, 1, 2, 3};
    SYSTEM_INFO sysInfo;

    GetSystemInfo(&sysInfo);
    printf("Number of processors: %lu\n\n", sysInfo.dwNumberOfProcessors);

    // 각 스레드를 특정 CPU에 바인딩
    for (int i = 0; i < 4 && i < (int)sysInfo.dwNumberOfProcessors; i++) {
        threads[i] = CreateThread(NULL, 0, AffinityThread, &ids[i],
                                  CREATE_SUSPENDED, NULL);

        // CPU 어피니티 설정 (비트 마스크)
        // 1 << i는 i번째 CPU에 바인딩
        DWORD_PTR mask = 1ULL << i;
        SetThreadAffinityMask(threads[i], mask);

        printf("Set thread %d affinity to CPU %d (mask: 0x%llx)\n",
               i, i, mask);

        ResumeThread(threads[i]);
    }

    // 대기
    WaitForMultipleObjects(4, threads, TRUE, INFINITE);

    for (int i = 0; i < 4; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

### POSIX: CPU Affinity

```c
#define _GNU_SOURCE
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

void* affinity_thread(void* arg) {
    int id = *(int*)arg;
    cpu_set_t cpuset;

    // 현재 어피니티 가져오기
    CPU_ZERO(&cpuset);
    pthread_getaffinity_np(pthread_self(), sizeof(cpu_set_t), &cpuset);

    printf("Thread %d:\n", id);
    printf("  Running on CPUs: ");
    for (int i = 0; i < CPU_SETSIZE; i++) {
        if (CPU_ISSET(i, &cpuset)) {
            printf("%d ", i);
        }
    }
    printf("\n");

    // 작업 수행
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return NULL;
}

int main() {
    pthread_t threads[4];
    pthread_attr_t attrs[4];
    int ids[4] = {0, 1, 2, 3};
    int num_cpus = sysconf(_SC_NPROCESSORS_ONLN);

    printf("Number of processors: %d\n\n", num_cpus);

    // 각 스레드를 특정 CPU에 바인딩
    for (int i = 0; i < 4 && i < num_cpus; i++) {
        cpu_set_t cpuset;

        pthread_attr_init(&attrs[i]);

        // CPU 어피니티 설정
        CPU_ZERO(&cpuset);
        CPU_SET(i, &cpuset);  // i번째 CPU에 바인딩

        pthread_attr_setaffinity_np(&attrs[i], sizeof(cpu_set_t), &cpuset);

        printf("Set thread %d affinity to CPU %d\n", i, i);

        pthread_create(&threads[i], &attrs[i], affinity_thread, &ids[i]);
    }

    // 대기
    for (int i = 0; i < 4 && i < num_cpus; i++) {
        pthread_join(threads[i], NULL);
        pthread_attr_destroy(&attrs[i]);
    }

    return 0;
}
```

### 어피니티 마스크 고급 사용

#### Windows: NUMA 인식 어피니티

```c
#include <windows.h>
#include <stdio.h>

void PrintNUMAInfo() {
    ULONG highestNodeNumber;

    if (GetNumaHighestNodeNumber(&highestNodeNumber)) {
        printf("NUMA nodes: %lu\n", highestNodeNumber + 1);

        for (ULONG node = 0; node <= highestNodeNumber; node++) {
            ULONGLONG processorMask;

            if (GetNumaNodeProcessorMask(node, &processorMask)) {
                printf("Node %lu processor mask: 0x%llx\n",
                       node, processorMask);
            }
        }
    } else {
        printf("Non-NUMA system\n");
    }
}

DWORD WINAPI NUMAThread(LPVOID lpParam) {
    int node = *(int*)lpParam;
    ULONGLONG processorMask;

    if (GetNumaNodeProcessorMask(node, &processorMask)) {
        SetThreadAffinityMask(GetCurrentThread(), processorMask);
        printf("Thread bound to NUMA node %d\n", node);
    }

    // 작업 수행
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return 0;
}

int main() {
    ULONG highestNodeNumber;

    printf("=== NUMA Information ===\n");
    PrintNUMAInfo();

    if (GetNumaHighestNodeNumber(&highestNodeNumber) &&
        highestNodeNumber > 0) {

        HANDLE threads[2];
        int nodes[2] = {0, 1};

        printf("\n=== Creating threads on different NUMA nodes ===\n");

        for (int i = 0; i <= (int)highestNodeNumber && i < 2; i++) {
            threads[i] = CreateThread(NULL, 0, NUMAThread,
                                     &nodes[i], 0, NULL);
        }

        WaitForMultipleObjects(2, threads, TRUE, INFINITE);

        for (int i = 0; i < 2; i++) {
            CloseHandle(threads[i]);
        }
    }

    return 0;
}
```

## 실시간 스케줄링

### Windows: 실시간 우선순위

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI RealtimeThread(LPVOID lpParam) {
    int id = *(int*)lpParam;
    LARGE_INTEGER frequency, start, end;

    QueryPerformanceFrequency(&frequency);
    QueryPerformanceCounter(&start);

    // 시간 제약이 있는 작업 시뮬레이션
    for (int i = 0; i < 10; i++) {
        // 정밀한 작업
        volatile long sum = 0;
        for (long j = 0; j < 10000000; j++) {
            sum += j;
        }

        // 1ms 대기
        Sleep(1);
    }

    QueryPerformanceCounter(&end);
    double elapsed = (double)(end.QuadPart - start.QuadPart) /
                     frequency.QuadPart * 1000.0;

    printf("Realtime thread %d: %.2f ms\n", id, elapsed);

    return 0;
}

int main() {
    HANDLE threads[2];
    int ids[2] = {1, 2};

    printf("=== Realtime Priority Test ===\n");
    printf("Warning: Realtime priority can affect system stability\n\n");

    // 프로세스 우선순위를 REALTIME으로 설정
    if (!SetPriorityClass(GetCurrentProcess(), REALTIME_PRIORITY_CLASS)) {
        printf("Failed to set REALTIME priority: %d\n", GetLastError());
        printf("Falling back to HIGH priority\n");
        SetPriorityClass(GetCurrentProcess(), HIGH_PRIORITY_CLASS);
    }

    // 실시간 스레드 생성
    for (int i = 0; i < 2; i++) {
        threads[i] = CreateThread(NULL, 0, RealtimeThread, &ids[i],
                                  CREATE_SUSPENDED, NULL);

        SetThreadPriority(threads[i], THREAD_PRIORITY_TIME_CRITICAL);
        ResumeThread(threads[i]);
    }

    WaitForMultipleObjects(2, threads, TRUE, INFINITE);

    for (int i = 0; i < 2; i++) {
        CloseHandle(threads[i]);
    }

    // 우선순위 복원
    SetPriorityClass(GetCurrentProcess(), NORMAL_PRIORITY_CLASS);

    return 0;
}
```

### POSIX: SCHED_FIFO와 SCHED_RR

```c
#include <pthread.h>
#include <sched.h>
#include <stdio.h>
#include <sys/time.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

void* realtime_thread(void* arg) {
    int id = *(int*)arg;
    struct timeval start, end;
    int policy;
    struct sched_param param;

    pthread_getschedparam(pthread_self(), &policy, &param);

    printf("Realtime thread %d:\n", id);
    printf("  Policy: %s\n",
           policy == SCHED_FIFO ? "SCHED_FIFO" :
           policy == SCHED_RR ? "SCHED_RR" : "OTHER");
    printf("  Priority: %d\n", param.sched_priority);

    gettimeofday(&start, NULL);

    // 시간 제약이 있는 작업 시뮬레이션
    for (int i = 0; i < 10; i++) {
        // 정밀한 작업
        volatile long sum = 0;
        for (long j = 0; j < 10000000; j++) {
            sum += j;
        }

        // 1ms 대기
        usleep(1000);
    }

    gettimeofday(&end, NULL);
    double elapsed = (end.tv_sec - start.tv_sec) * 1000.0 +
                     (end.tv_usec - start.tv_usec) / 1000.0;

    printf("Realtime thread %d: %.2f ms\n", id, elapsed);

    return NULL;
}

int main() {
    pthread_t threads[2];
    pthread_attr_t attrs[2];
    int ids[2] = {1, 2};
    struct sched_param param;

    printf("=== POSIX Realtime Scheduling Test ===\n");
    printf("Note: Requires root privileges (sudo)\n\n");

    // SCHED_FIFO 우선순위 범위 확인
    int min_prio = sched_get_priority_min(SCHED_FIFO);
    int max_prio = sched_get_priority_max(SCHED_FIFO);

    printf("SCHED_FIFO priority range: %d - %d\n\n", min_prio, max_prio);

    // 스레드 1: SCHED_FIFO
    pthread_attr_init(&attrs[0]);
    pthread_attr_setschedpolicy(&attrs[0], SCHED_FIFO);
    param.sched_priority = max_prio - 1;
    pthread_attr_setschedparam(&attrs[0], &param);
    pthread_attr_setinheritsched(&attrs[0], PTHREAD_EXPLICIT_SCHED);

    // 스레드 2: SCHED_RR
    pthread_attr_init(&attrs[1]);
    pthread_attr_setschedpolicy(&attrs[1], SCHED_RR);
    param.sched_priority = max_prio - 1;
    pthread_attr_setschedparam(&attrs[1], &param);
    pthread_attr_setinheritsched(&attrs[1], PTHREAD_EXPLICIT_SCHED);

    // 스레드 생성
    for (int i = 0; i < 2; i++) {
        if (pthread_create(&threads[i], &attrs[i],
                          realtime_thread, &ids[i]) != 0) {
            printf("Failed to create realtime thread %d: %s\n",
                   i, strerror(errno));
            printf("Try running with sudo\n");

            // 기본 속성으로 재시도
            pthread_create(&threads[i], NULL, realtime_thread, &ids[i]);
        }
    }

    // 대기
    for (int i = 0; i < 2; i++) {
        pthread_join(threads[i], NULL);
        pthread_attr_destroy(&attrs[i]);
    }

    return 0;
}
```

## 성능 측정과 분석

### 컨텍스트 스위치 측정

#### Windows

```c
#include <windows.h>
#include <stdio.h>

void MeasureContextSwitch() {
    HANDLE hEvent1, hEvent2;
    LARGE_INTEGER frequency, start, end;
    const int ITERATIONS = 10000;

    hEvent1 = CreateEvent(NULL, FALSE, FALSE, NULL);
    hEvent2 = CreateEvent(NULL, FALSE, FALSE, NULL);

    QueryPerformanceFrequency(&frequency);

    // 워밍업
    for (int i = 0; i < 1000; i++) {
        SetEvent(hEvent1);
        WaitForSingleObject(hEvent1, 0);
    }

    QueryPerformanceCounter(&start);

    for (int i = 0; i < ITERATIONS; i++) {
        SetEvent(hEvent1);
        WaitForSingleObject(hEvent1, 0);
    }

    QueryPerformanceCounter(&end);

    double elapsed = (double)(end.QuadPart - start.QuadPart) /
                     frequency.QuadPart * 1000000.0;  // 마이크로초

    printf("Average event signal/wait time: %.2f microseconds\n",
           elapsed / ITERATIONS);

    CloseHandle(hEvent1);
    CloseHandle(hEvent2);
}

int main() {
    printf("=== Context Switch Performance ===\n");
    MeasureContextSwitch();
    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <sys/time.h>
#include <stdbool.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
bool ready = false;

void measure_context_switch() {
    struct timeval start, end;
    const int ITERATIONS = 10000;

    // 워밍업
    for (int i = 0; i < 1000; i++) {
        pthread_mutex_lock(&mutex);
        pthread_mutex_unlock(&mutex);
    }

    gettimeofday(&start, NULL);

    for (int i = 0; i < ITERATIONS; i++) {
        pthread_mutex_lock(&mutex);
        ready = true;
        pthread_cond_signal(&cond);
        pthread_mutex_unlock(&mutex);

        pthread_mutex_lock(&mutex);
        ready = false;
        pthread_mutex_unlock(&mutex);
    }

    gettimeofday(&end, NULL);

    double elapsed = (end.tv_sec - start.tv_sec) * 1000000.0 +
                     (end.tv_usec - start.tv_usec);

    printf("Average mutex lock/unlock time: %.2f microseconds\n",
           elapsed / ITERATIONS);
}

int main() {
    printf("=== Context Switch Performance ===\n");
    measure_context_switch();

    pthread_mutex_destroy(&mutex);
    pthread_cond_destroy(&cond);

    return 0;
}
```

## 실용적 권장사항

### 1. 기본 우선순위 사용

대부분의 경우 기본 우선순위가 적절합니다:

```c
// Windows: NORMAL priority class, NORMAL thread priority
// POSIX: SCHED_OTHER with nice 0
```

### 2. 우선순위 역전 방지

```c
// 나쁜 예: 낮은 우선순위 스레드가 높은 우선순위 스레드가 필요한 락을 보유
// 좋은 예: 우선순위 상속 뮤텍스 사용 (POSIX)

pthread_mutexattr_t attr;
pthread_mutex_t mutex;

pthread_mutexattr_init(&attr);
pthread_mutexattr_setprotocol(&attr, PTHREAD_PRIO_INHERIT);
pthread_mutex_init(&mutex, &attr);
```

### 3. CPU 어피니티 신중히 사용

```c
// 어피니티는 특정 상황에서만 유용:
// - NUMA 시스템에서 메모리 지역성
// - 캐시 친화적인 작업
// - 실시간 요구사항

// 일반적으로 OS에게 맡기는 것이 더 좋음
```

### 4. 실시간 우선순위 주의

```c
// 실시간 우선순위는 시스템 불안정을 초래할 수 있음
// 다음 경우에만 사용:
// - 오디오/비디오 처리
// - 산업 제어 시스템
// - 시간 제약이 엄격한 작업

// 항상 타임아웃과 에러 처리 포함
```

### 5. 크로스 플랫폼 래퍼

```c
// priority_wrapper.h
#ifdef _WIN32
    #include <windows.h>

    typedef enum {
        PRIO_LOW,
        PRIO_NORMAL,
        PRIO_HIGH,
        PRIO_REALTIME
    } thread_priority_t;

    static inline void set_thread_priority(thread_priority_t prio) {
        int win_prio;
        switch (prio) {
            case PRIO_LOW:
                win_prio = THREAD_PRIORITY_BELOW_NORMAL;
                break;
            case PRIO_HIGH:
                win_prio = THREAD_PRIORITY_ABOVE_NORMAL;
                break;
            case PRIO_REALTIME:
                win_prio = THREAD_PRIORITY_TIME_CRITICAL;
                break;
            default:
                win_prio = THREAD_PRIORITY_NORMAL;
        }
        SetThreadPriority(GetCurrentThread(), win_prio);
    }
#else
    #include <pthread.h>
    #include <sys/resource.h>

    typedef enum {
        PRIO_LOW,
        PRIO_NORMAL,
        PRIO_HIGH,
        PRIO_REALTIME
    } thread_priority_t;

    static inline void set_thread_priority(thread_priority_t prio) {
        int nice_val;
        switch (prio) {
            case PRIO_LOW:
                nice_val = 10;
                break;
            case PRIO_HIGH:
                nice_val = -10;
                break;
            case PRIO_REALTIME:
                nice_val = -20;
                break;
            default:
                nice_val = 0;
        }
        setpriority(PRIO_PROCESS, 0, nice_val);
    }
#endif
```

## 요약

### 주요 차이점

| 항목 | Windows | POSIX |
|------|---------|-------|
| 우선순위 모델 | 2단계 (프로세스 + 스레드) | nice + sched_priority |
| 동적 조정 | 자동 부스팅 | 정책에 따라 다름 |
| 실시간 지원 | REALTIME priority class | SCHED_FIFO/RR |
| 어피니티 | 비트 마스크 | cpu_set_t |
| 권한 요구사항 | 일부 작업만 | 실시간은 root 필요 |

### 선택 가이드

| 요구사항 | Windows 권장 | POSIX 권장 |
|---------|-------------|-----------|
| 일반 애플리케이션 | NORMAL priority | SCHED_OTHER |
| 백그라운드 작업 | IDLE/LOWEST | SCHED_IDLE 또는 nice 19 |
| UI 반응성 | ABOVE_NORMAL | nice -5 |
| 실시간 처리 | HIGH + TIME_CRITICAL | SCHED_FIFO |
| CPU 바인딩 | SetThreadAffinityMask | pthread_setaffinity_np |

적절한 스케줄링 설정은 애플리케이션 성능과 사용자 경험에 큰 영향을 미칩니다. 항상 프로파일링을 통해 실제 효과를 검증하세요.
