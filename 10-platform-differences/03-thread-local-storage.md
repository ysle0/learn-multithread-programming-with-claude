# Thread Local Storage (TLS): Windows vs POSIX

## 목차
1. [개요](#개요)
2. [동적 TLS](#동적-tls)
3. [정적 TLS](#정적-tls)
4. [TLS 사용 패턴](#tls-사용-패턴)
5. [성능 고려사항](#성능-고려사항)
6. [실용적 권장사항](#실용적-권장사항)

## 개요

Thread Local Storage(TLS)는 각 스레드가 고유한 데이터 복사본을 가질 수 있게 하는 메커니즘입니다. 전역 변수처럼 보이지만 각 스레드에서 독립적인 값을 가집니다.

### TLS의 두 가지 방식

1. **동적 TLS (Dynamic TLS)**: 런타임에 할당
   - Windows: `TlsAlloc`, `TlsGetValue`, `TlsSetValue`, `TlsFree`
   - POSIX: `pthread_key_create`, `pthread_getspecific`, `pthread_setspecific`, `pthread_key_delete`

2. **정적 TLS (Static TLS)**: 컴파일 타임에 할당
   - Windows: `__declspec(thread)`
   - POSIX: `__thread` (또는 C11 `_Thread_local`)

### 비교 표

| 항목 | Windows (동적) | POSIX (동적) | Windows (정적) | POSIX (정적) |
|------|---------------|-------------|---------------|-------------|
| 할당 | `TlsAlloc` | `pthread_key_create` | `__declspec(thread)` | `__thread` |
| 설정 | `TlsSetValue` | `pthread_setspecific` | 직접 할당 | 직접 할당 |
| 읽기 | `TlsGetValue` | `pthread_getspecific` | 직접 읽기 | 직접 읽기 |
| 해제 | `TlsFree` | `pthread_key_delete` | 자동 | 자동 |
| 성능 | 중간 | 중간 | 빠름 | 빠름 |

## 동적 TLS

### Windows: TLS API

```c
#include <windows.h>
#include <stdio.h>

// TLS 인덱스 (전역)
DWORD g_tlsIndex;

typedef struct {
    int thread_id;
    int counter;
    char name[64];
} ThreadData;

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    // TLS에 저장할 데이터 할당
    ThreadData* data = (ThreadData*)malloc(sizeof(ThreadData));
    data->thread_id = id;
    data->counter = 0;
    snprintf(data->name, sizeof(data->name), "Thread_%d", id);

    // TLS에 데이터 저장
    if (!TlsSetValue(g_tlsIndex, data)) {
        printf("TlsSetValue failed: %d\n", GetLastError());
        free(data);
        return 1;
    }

    // 작업 수행
    for (int i = 0; i < 5; i++) {
        // TLS에서 데이터 읽기
        ThreadData* myData = (ThreadData*)TlsGetValue(g_tlsIndex);

        if (myData != NULL) {
            myData->counter++;
            printf("%s: counter = %d\n", myData->name, myData->counter);
        }

        Sleep(100);
    }

    // TLS에서 데이터 가져와서 해제
    ThreadData* myData = (ThreadData*)TlsGetValue(g_tlsIndex);
    if (myData != NULL) {
        printf("%s: Final counter = %d\n", myData->name, myData->counter);
        free(myData);
    }

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    // TLS 인덱스 할당
    g_tlsIndex = TlsAlloc();
    if (g_tlsIndex == TLS_OUT_OF_INDEXES) {
        printf("TlsAlloc failed\n");
        return 1;
    }

    printf("TLS index allocated: %lu\n", g_tlsIndex);

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
    }

    // 스레드 대기
    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    // TLS 인덱스 해제
    TlsFree(g_tlsIndex);

    return 0;
}
```

### POSIX: pthread_key_t

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

// TLS 키 (전역)
pthread_key_t g_tls_key;

typedef struct {
    int thread_id;
    int counter;
    char name[64];
} ThreadData;

// TLS 정리 함수 (스레드 종료 시 자동 호출)
void cleanup_thread_data(void* data) {
    ThreadData* thread_data = (ThreadData*)data;
    printf("Cleanup: %s (counter = %d)\n",
           thread_data->name, thread_data->counter);
    free(thread_data);
}

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    // TLS에 저장할 데이터 할당
    ThreadData* data = (ThreadData*)malloc(sizeof(ThreadData));
    data->thread_id = id;
    data->counter = 0;
    snprintf(data->name, sizeof(data->name), "Thread_%d", id);

    // TLS에 데이터 저장
    pthread_setspecific(g_tls_key, data);

    // 작업 수행
    for (int i = 0; i < 5; i++) {
        // TLS에서 데이터 읽기
        ThreadData* myData = (ThreadData*)pthread_getspecific(g_tls_key);

        if (myData != NULL) {
            myData->counter++;
            printf("%s: counter = %d\n", myData->name, myData->counter);
        }

        usleep(100000);
    }

    // TLS에서 데이터 가져오기
    ThreadData* myData = (ThreadData*)pthread_getspecific(g_tls_key);
    if (myData != NULL) {
        printf("%s: Final counter = %d\n", myData->name, myData->counter);
    }

    // 정리 함수가 자동으로 호출됨
    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    // TLS 키 생성 (정리 함수 등록)
    if (pthread_key_create(&g_tls_key, cleanup_thread_data) != 0) {
        printf("pthread_key_create failed\n");
        return 1;
    }

    printf("TLS key created\n");

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    // 스레드 대기
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    // TLS 키 삭제
    pthread_key_delete(g_tls_key);

    return 0;
}
```

### 동적 TLS의 장단점

#### 장점
- 런타임에 필요한 만큼 할당 가능
- 정리 함수 지원 (POSIX)
- DLL/공유 라이브러리에서 안전하게 사용 가능

#### 단점
- 함수 호출 오버헤드
- 명시적인 초기화/정리 필요
- 키 개수 제한 (플랫폼 의존적)

### TLS 정리 함수 (Destructor) 비교

#### Windows: DllMain 사용

Windows는 POSIX처럼 자동 정리 함수를 제공하지 않으므로 DllMain을 사용해야 합니다.

```c
#include <windows.h>
#include <stdio.h>

DWORD g_tlsIndex;

typedef struct {
    int value;
    char* buffer;
} TlsData;

void CleanupTlsData(TlsData* data) {
    if (data != NULL) {
        printf("Cleaning up TLS data (value = %d)\n", data->value);
        if (data->buffer != NULL) {
            free(data->buffer);
        }
        free(data);
    }
}

// DLL 진입점 (EXE에서도 사용 가능)
BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved) {
    TlsData* data;

    switch (fdwReason) {
        case DLL_PROCESS_ATTACH:
            g_tlsIndex = TlsAlloc();
            if (g_tlsIndex == TLS_OUT_OF_INDEXES) {
                return FALSE;
            }
            // No break: Initialize TLS for first thread

        case DLL_THREAD_ATTACH:
            // 새 스레드용 TLS 초기화
            break;

        case DLL_THREAD_DETACH:
            // 스레드 종료 시 TLS 정리
            data = (TlsData*)TlsGetValue(g_tlsIndex);
            CleanupTlsData(data);
            break;

        case DLL_PROCESS_DETACH:
            // 프로세스 종료 시 정리
            data = (TlsData*)TlsGetValue(g_tlsIndex);
            CleanupTlsData(data);
            TlsFree(g_tlsIndex);
            break;
    }

    return TRUE;
}

// 또는 atexit을 사용한 수동 정리
void manual_cleanup() {
    TlsData* data = (TlsData*)TlsGetValue(g_tlsIndex);
    CleanupTlsData(data);
}

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    TlsData* data = (TlsData*)malloc(sizeof(TlsData));
    data->value = *(int*)lpParam;
    data->buffer = (char*)malloc(256);

    TlsSetValue(g_tlsIndex, data);

    // 작업 수행...

    // 수동으로 정리 (DllMain이 없을 경우)
    // manual_cleanup();

    return 0;
}
```

#### POSIX: pthread_key_create의 destructor

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

pthread_key_t g_tls_key;

typedef struct {
    int value;
    char* buffer;
} TlsData;

// 자동 정리 함수
void cleanup_tls_data(void* data) {
    TlsData* tls_data = (TlsData*)data;
    if (tls_data != NULL) {
        printf("Cleaning up TLS data (value = %d)\n", tls_data->value);
        if (tls_data->buffer != NULL) {
            free(tls_data->buffer);
        }
        free(tls_data);
    }
}

void* worker_thread(void* arg) {
    TlsData* data = (TlsData*)malloc(sizeof(TlsData));
    data->value = *(int*)arg;
    data->buffer = (char*)malloc(256);

    pthread_setspecific(g_tls_key, data);

    // 작업 수행...

    // 정리 함수가 스레드 종료 시 자동으로 호출됨
    return NULL;
}

int main() {
    pthread_t thread;
    int value = 42;

    // 정리 함수와 함께 TLS 키 생성
    pthread_key_create(&g_tls_key, cleanup_tls_data);

    pthread_create(&thread, NULL, worker_thread, &value);
    pthread_join(thread, NULL);

    pthread_key_delete(g_tls_key);

    return 0;
}
```

## 정적 TLS

### Windows: __declspec(thread)

```c
#include <windows.h>
#include <stdio.h>

// 정적 TLS 변수
__declspec(thread) int g_threadCounter = 0;
__declspec(thread) char g_threadName[64] = "";

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    // 스레드 로컬 변수 초기화
    g_threadCounter = 0;
    snprintf(g_threadName, sizeof(g_threadName), "Thread_%d", id);

    // 각 스레드는 독립적인 복사본을 가짐
    for (int i = 0; i < 5; i++) {
        g_threadCounter++;
        printf("%s: counter = %d\n", g_threadName, g_threadCounter);
        Sleep(100);
    }

    printf("%s: Final counter = %d\n", g_threadName, g_threadCounter);

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    // 메인 스레드의 TLS 변수
    g_threadCounter = 100;
    snprintf(g_threadName, sizeof(g_threadName), "MainThread");

    printf("%s: Initial counter = %d\n", g_threadName, g_threadCounter);

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
    }

    // 메인 스레드의 카운터는 독립적
    printf("%s: Counter still = %d\n", g_threadName, g_threadCounter);

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    printf("%s: Final counter = %d\n", g_threadName, g_threadCounter);

    return 0;
}
```

### POSIX: __thread

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>

// 정적 TLS 변수
__thread int g_thread_counter = 0;
__thread char g_thread_name[64] = "";

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    // 스레드 로컬 변수 초기화
    g_thread_counter = 0;
    snprintf(g_thread_name, sizeof(g_thread_name), "Thread_%d", id);

    // 각 스레드는 독립적인 복사본을 가짐
    for (int i = 0; i < 5; i++) {
        g_thread_counter++;
        printf("%s: counter = %d\n", g_thread_name, g_thread_counter);
        usleep(100000);
    }

    printf("%s: Final counter = %d\n", g_thread_name, g_thread_counter);

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    // 메인 스레드의 TLS 변수
    g_thread_counter = 100;
    snprintf(g_thread_name, sizeof(g_thread_name), "MainThread");

    printf("%s: Initial counter = %d\n", g_thread_name, g_thread_counter);

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    // 메인 스레드의 카운터는 독립적
    printf("%s: Counter still = %d\n", g_thread_name, g_thread_counter);

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("%s: Final counter = %d\n", g_thread_name, g_thread_counter);

    return 0;
}
```

### C11 표준: _Thread_local

C11 표준은 `_Thread_local` 키워드를 제공합니다.

```c
#include <threads.h>
#include <stdio.h>

// C11 표준 TLS
_Thread_local int counter = 0;

// 또는 편의 매크로 사용
// thread_local int counter = 0;  // <threads.h> 필요

int thread_func(void* arg) {
    int id = *(int*)arg;

    counter = id * 10;

    printf("Thread %d: counter = %d\n", id, counter);

    return 0;
}

int main() {
    thrd_t threads[3];
    int ids[3] = {1, 2, 3};

    counter = 100;
    printf("Main: counter = %d\n", counter);

    for (int i = 0; i < 3; i++) {
        thrd_create(&threads[i], thread_func, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        thrd_join(threads[i], NULL);
    }

    printf("Main: counter still = %d\n", counter);

    return 0;
}
```

### 정적 TLS의 장단점

#### 장점
- 빠른 접근 속도 (함수 호출 없음)
- 간단한 문법 (일반 변수처럼 사용)
- 자동 초기화 및 정리

#### 단점
- 컴파일 타임에 결정
- DLL 로딩 시 제한사항 (Windows)
- 메모리 오버헤드 (모든 스레드에 할당)

### DLL과 정적 TLS 주의사항

#### Windows: LoadLibrary 제한

```c
// DLL 코드
__declspec(dllexport) __declspec(thread) int dll_tls_var = 0;

// 메인 프로그램
HMODULE hDll = LoadLibrary("mydll.dll");  // 위험!

// Windows XP 이전: 런타임에 로드된 DLL의 TLS 변수는 제대로 초기화되지 않음
// Vista 이상: 지원되지만 성능 오버헤드 있음

// 해결책: 동적 TLS 사용 또는 암시적 링크
```

#### POSIX: dlopen 동작

```c
// 공유 라이브러리 코드
__thread int lib_tls_var = 0;

// 메인 프로그램
void* handle = dlopen("libmylib.so", RTLD_NOW);

// 대부분의 POSIX 시스템에서 정상 동작
// 그러나 이식성을 위해서는 동적 TLS 권장
```

## TLS 사용 패턴

### 1. 스레드별 캐시

#### Windows

```c
#include <windows.h>
#include <stdio.h>

__declspec(thread) char g_cache[1024] = "";
__declspec(thread) BOOL g_cacheValid = FALSE;

DWORD WINAPI CacheWorker(LPVOID lpParam) {
    int id = *(int*)lpParam;

    // 첫 번째 호출: 캐시 초기화
    if (!g_cacheValid) {
        snprintf(g_cache, sizeof(g_cache),
                 "Thread %d cache data", id);
        g_cacheValid = TRUE;
        printf("Thread %d: Cache initialized\n", id);
    }

    // 캐시 사용
    for (int i = 0; i < 3; i++) {
        printf("Thread %d: Using cache: %s\n", id, g_cache);
        Sleep(100);
    }

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, CacheWorker, &ids[i], 0, NULL);
    }

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>
#include <string.h>

__thread char g_cache[1024] = "";
__thread bool g_cache_valid = false;

void* cache_worker(void* arg) {
    int id = *(int*)arg;

    // 첫 번째 호출: 캐시 초기화
    if (!g_cache_valid) {
        snprintf(g_cache, sizeof(g_cache),
                 "Thread %d cache data", id);
        g_cache_valid = true;
        printf("Thread %d: Cache initialized\n", id);
    }

    // 캐시 사용
    for (int i = 0; i < 3; i++) {
        printf("Thread %d: Using cache: %s\n", id, g_cache);
        usleep(100000);
    }

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, cache_worker, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### 2. 스레드별 에러 코드

#### Windows (errno 스타일)

```c
#include <windows.h>
#include <stdio.h>

__declspec(thread) int g_lastError = 0;

void SetMyError(int error) {
    g_lastError = error;
}

int GetMyError() {
    return g_lastError;
}

BOOL ProcessData(int value) {
    if (value < 0) {
        SetMyError(100);  // 커스텀 에러 코드
        return FALSE;
    }
    if (value > 1000) {
        SetMyError(200);
        return FALSE;
    }
    return TRUE;
}

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    // 각 스레드는 독립적인 에러 상태를 가짐
    if (!ProcessData(id * 100)) {
        printf("Thread %d: Error %d\n", id, GetMyError());
    } else {
        printf("Thread %d: Success\n", id);
    }

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, -5, 15};  // 하나는 에러 발생

    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
    }

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

#### POSIX (errno는 이미 TLS)

```c
#include <pthread.h>
#include <stdio.h>
#include <errno.h>

__thread int g_last_error = 0;

void set_my_error(int error) {
    g_last_error = error;
}

int get_my_error() {
    return g_last_error;
}

bool process_data(int value) {
    if (value < 0) {
        set_my_error(100);
        return false;
    }
    if (value > 1000) {
        set_my_error(200);
        return false;
    }
    return true;
}

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    // 각 스레드는 독립적인 에러 상태를 가짐
    if (!process_data(id * 100)) {
        printf("Thread %d: Error %d\n", id, get_my_error());
    } else {
        printf("Thread %d: Success\n", id);
    }

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, -5, 15};

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### 3. 스레드별 난수 생성기

#### Windows

```c
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>

__declspec(thread) unsigned int g_randSeed = 0;

void InitThreadRandom() {
    // 스레드 ID와 시간을 조합하여 시드 생성
    g_randSeed = GetCurrentThreadId() ^ (unsigned int)time(NULL);
}

int ThreadRand() {
    // 간단한 LCG (Linear Congruential Generator)
    g_randSeed = g_randSeed * 1103515245 + 12345;
    return (g_randSeed / 65536) % 32768;
}

DWORD WINAPI RandomWorker(LPVOID lpParam) {
    int id = *(int*)lpParam;

    InitThreadRandom();

    for (int i = 0; i < 5; i++) {
        int random = ThreadRand();
        printf("Thread %d: Random = %d\n", id, random);
        Sleep(100);
    }

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, RandomWorker, &ids[i], 0, NULL);
    }

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <unistd.h>

__thread unsigned int g_rand_seed = 0;

void init_thread_random() {
    g_rand_seed = (unsigned int)pthread_self() ^ (unsigned int)time(NULL);
}

int thread_rand() {
    g_rand_seed = g_rand_seed * 1103515245 + 12345;
    return (g_rand_seed / 65536) % 32768;
}

void* random_worker(void* arg) {
    int id = *(int*)arg;

    init_thread_random();

    for (int i = 0; i < 5; i++) {
        int random = thread_rand();
        printf("Thread %d: Random = %d\n", id, random);
        usleep(100000);
    }

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, random_worker, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### 4. 스레드별 메모리 풀

#### 동적 TLS 사용 (POSIX 예제)

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define POOL_SIZE 10
#define BLOCK_SIZE 256

typedef struct {
    void* blocks[POOL_SIZE];
    int next_free;
} MemoryPool;

pthread_key_t g_pool_key;

void cleanup_pool(void* data) {
    MemoryPool* pool = (MemoryPool*)data;
    printf("Cleaning up memory pool\n");

    for (int i = 0; i < POOL_SIZE; i++) {
        if (pool->blocks[i] != NULL) {
            free(pool->blocks[i]);
        }
    }
    free(pool);
}

MemoryPool* get_thread_pool() {
    MemoryPool* pool = (MemoryPool*)pthread_getspecific(g_pool_key);

    if (pool == NULL) {
        // 첫 호출: 풀 생성
        pool = (MemoryPool*)malloc(sizeof(MemoryPool));
        memset(pool, 0, sizeof(MemoryPool));
        pool->next_free = 0;

        pthread_setspecific(g_pool_key, pool);
        printf("Memory pool created for thread\n");
    }

    return pool;
}

void* pool_alloc() {
    MemoryPool* pool = get_thread_pool();

    if (pool->next_free >= POOL_SIZE) {
        return NULL;  // 풀이 가득 참
    }

    void* block = malloc(BLOCK_SIZE);
    pool->blocks[pool->next_free++] = block;

    return block;
}

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    // 스레드별 메모리 풀에서 할당
    for (int i = 0; i < 5; i++) {
        void* ptr = pool_alloc();
        if (ptr != NULL) {
            sprintf((char*)ptr, "Thread %d, allocation %d", id, i);
            printf("%s\n", (char*)ptr);
        }
    }

    // 정리 함수가 자동으로 호출됨
    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    pthread_key_create(&g_pool_key, cleanup_pool);

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_key_delete(&g_pool_key);

    return 0;
}
```

## 성능 고려사항

### 접근 속도 비교

```c
#include <stdio.h>
#include <time.h>

#ifdef _WIN32
    #include <windows.h>
    DWORD g_tls_index;
    __declspec(thread) int g_static_tls = 0;
#else
    #include <pthread.h>
    pthread_key_t g_tls_key;
    __thread int g_static_tls = 0;
#endif

#define ITERATIONS 10000000

int main() {
    clock_t start, end;
    double cpu_time;
    volatile int value;

    #ifdef _WIN32
        g_tls_index = TlsAlloc();
        TlsSetValue(g_tls_index, (void*)42);
    #else
        pthread_key_create(&g_tls_key, NULL);
        pthread_setspecific(g_tls_key, (void*)42);
    #endif

    // 정적 TLS 벤치마크
    printf("Benchmarking static TLS...\n");
    start = clock();

    for (int i = 0; i < ITERATIONS; i++) {
        value = g_static_tls;
        g_static_tls = i;
    }

    end = clock();
    cpu_time = ((double)(end - start)) / CLOCKS_PER_SEC;
    printf("Static TLS: %.3f seconds (%.2f ns per access)\n",
           cpu_time, (cpu_time / ITERATIONS) * 1000000000);

    // 동적 TLS 벤치마크
    printf("\nBenchmarking dynamic TLS...\n");
    start = clock();

    for (int i = 0; i < ITERATIONS; i++) {
        #ifdef _WIN32
            value = (int)(size_t)TlsGetValue(g_tls_index);
            TlsSetValue(g_tls_index, (void*)(size_t)i);
        #else
            value = (int)(size_t)pthread_getspecific(g_tls_key);
            pthread_setspecific(g_tls_key, (void*)(size_t)i);
        #endif
    }

    end = clock();
    cpu_time = ((double)(end - start)) / CLOCKS_PER_SEC;
    printf("Dynamic TLS: %.3f seconds (%.2f ns per access)\n",
           cpu_time, (cpu_time / ITERATIONS) * 1000000000);

    #ifdef _WIN32
        TlsFree(g_tls_index);
    #else
        pthread_key_delete(g_tls_key);
    #endif

    return 0;
}
```

### 일반적인 성능 결과

| 작업 | 정적 TLS | 동적 TLS | 일반 전역 변수 |
|------|---------|---------|--------------|
| 읽기 | ~0.5-2 ns | ~5-20 ns | ~0.3-1 ns |
| 쓰기 | ~0.5-2 ns | ~10-30 ns | ~0.3-1 ns |
| 메모리 오버헤드 | 스레드당 고정 | 사용한 만큼 | 단일 복사본 |

### 최적화 팁

1. **자주 접근하는 데이터는 정적 TLS 사용**
```c
__declspec(thread) int fast_counter = 0;  // Windows
__thread int fast_counter = 0;            // POSIX
```

2. **크고 복잡한 구조체는 동적 TLS 사용**
```c
// 동적 할당으로 메모리 절약
pthread_key_t g_large_data_key;
// 필요한 스레드만 할당받음
```

3. **TLS를 로컬 변수에 캐시**
```c
// 나쁜 예
for (int i = 0; i < 1000000; i++) {
    TlsGetValue(g_tls_index);  // 매번 함수 호출
}

// 좋은 예
void* cached = TlsGetValue(g_tls_index);
for (int i = 0; i < 1000000; i++) {
    // cached 사용
}
```

## 실용적 권장사항

### 언제 정적 TLS를 사용할까?

1. 빠른 접근이 중요할 때
2. 컴파일 타임에 필요성을 알 수 있을 때
3. 모든 스레드가 사용할 가능성이 높을 때

```c
__declspec(thread) int thread_counter = 0;     // Windows
__thread int thread_counter = 0;               // POSIX
_Thread_local int thread_counter = 0;          // C11
```

### 언제 동적 TLS를 사용할까?

1. 라이브러리 코드에서 (DLL 로딩 문제 회피)
2. 일부 스레드만 사용할 때 (메모리 절약)
3. 정리 함수가 필요할 때 (POSIX)
4. 런타임에 결정될 때

```c
// Windows
DWORD tls_index = TlsAlloc();
TlsSetValue(tls_index, data);

// POSIX
pthread_key_t tls_key;
pthread_key_create(&tls_key, cleanup_func);
pthread_setspecific(tls_key, data);
```

### 크로스 플랫폼 TLS 래퍼

```c
// tls_wrapper.h
#ifdef _WIN32
    #include <windows.h>
    typedef DWORD tls_key_t;
    typedef void (*tls_destructor_t)(void*);

    #define TLS_ALLOC(key) ((key) = TlsAlloc(), (key) != TLS_OUT_OF_INDEXES)
    #define TLS_SET(key, value) TlsSetValue((key), (value))
    #define TLS_GET(key) TlsGetValue(key)
    #define TLS_FREE(key) TlsFree(key)

    // 정적 TLS
    #define THREAD_LOCAL __declspec(thread)
#else
    #include <pthread.h>
    typedef pthread_key_t tls_key_t;
    typedef void (*tls_destructor_t)(void*);

    static inline int tls_create_with_destructor(tls_key_t* key,
                                                   tls_destructor_t destructor) {
        return pthread_key_create(key, destructor) == 0;
    }

    #define TLS_ALLOC(key) tls_create_with_destructor(&(key), NULL)
    #define TLS_SET(key, value) (pthread_setspecific((key), (value)) == 0)
    #define TLS_GET(key) pthread_getspecific(key)
    #define TLS_FREE(key) (pthread_key_delete(key) == 0)

    // 정적 TLS
    #define THREAD_LOCAL __thread
#endif

// 사용 예
THREAD_LOCAL int my_counter = 0;

tls_key_t my_key;

void init_tls() {
    TLS_ALLOC(my_key);
}

void use_tls() {
    TLS_SET(my_key, (void*)123);
    void* value = TLS_GET(my_key);
}

void cleanup_tls() {
    TLS_FREE(my_key);
}
```

### C++ RAII 래퍼

```cpp
#ifdef _WIN32
    #include <windows.h>
#else
    #include <pthread.h>
#endif

template<typename T>
class ThreadLocal {
private:
    #ifdef _WIN32
        DWORD key_;
    #else
        pthread_key_t key_;
    #endif

    static void destructor(void* ptr) {
        delete static_cast<T*>(ptr);
    }

public:
    ThreadLocal() {
        #ifdef _WIN32
            key_ = TlsAlloc();
        #else
            pthread_key_create(&key_, destructor);
        #endif
    }

    ~ThreadLocal() {
        #ifdef _WIN32
            TlsFree(key_);
        #else
            pthread_key_delete(key_);
        #endif
    }

    T* get() {
        #ifdef _WIN32
            return static_cast<T*>(TlsGetValue(key_));
        #else
            return static_cast<T*>(pthread_getspecific(key_));
        #endif
    }

    void set(T* value) {
        #ifdef _WIN32
            TlsSetValue(key_, value);
        #else
            pthread_setspecific(key_, value);
        #endif
    }

    T& operator*() {
        T* ptr = get();
        if (!ptr) {
            ptr = new T();
            set(ptr);
        }
        return *ptr;
    }

    T* operator->() {
        return &(**this);
    }
};

// 사용 예
ThreadLocal<int> thread_counter;

void worker_function() {
    *thread_counter = 42;
    printf("Counter: %d\n", *thread_counter);
}
```

## 요약

### 선택 가이드

| 요구사항 | 권장 방법 |
|---------|----------|
| 최고 성능 | 정적 TLS (`__declspec(thread)` / `__thread`) |
| 라이브러리 코드 | 동적 TLS (TlsAlloc / pthread_key_create) |
| 자동 정리 필요 | POSIX 동적 TLS (destructor 지원) |
| 크로스 플랫폼 | C++11 `thread_local` 또는 래퍼 클래스 |
| C11 표준 준수 | `_Thread_local` |

### 주의사항

1. **메모리 누수**: 동적 TLS 사용 시 정리 함수 필수
2. **DLL 로딩**: Windows에서 정적 TLS는 LoadLibrary와 함께 사용 시 주의
3. **초기화**: 정적 TLS는 0으로 초기화됨, 복잡한 초기화는 수동으로
4. **스레드 풀**: 스레드 재사용 시 TLS 값 리셋 필요

TLS는 스레드 안전한 코드를 작성하는 강력한 도구이지만, 과도한 사용은 메모리 낭비와 복잡성 증가를 초래할 수 있습니다. 필요한 곳에만 신중하게 사용하세요.
