# 동기화 프리미티브: Windows vs POSIX

## 목차
1. [개요](#개요)
2. [Mutex 비교](#mutex-비교)
3. [Condition Variable 비교](#condition-variable-비교)
4. [세마포어 비교](#세마포어-비교)
5. [Read-Write Lock 비교](#read-write-lock-비교)
6. [여러 객체 대기](#여러-객체-대기)
7. [성능 비교](#성능-비교)
8. [실용적 권장사항](#실용적-권장사항)

## 개요

동기화 프리미티브는 멀티스레드 프로그래밍의 핵심입니다. Windows와 POSIX는 유사한 개념을 제공하지만 API와 구현이 다릅니다.

### 주요 차이점 요약

| 프리미티브 | Windows | POSIX | 주요 차이 |
|-----------|---------|-------|----------|
| Mutex | `CRITICAL_SECTION` (빠름) | `pthread_mutex_t` | Windows는 두 종류 제공 |
| | `Mutex` (프로세스 간) | | |
| Condition Variable | `CONDITION_VARIABLE` | `pthread_cond_t` | 유사한 개념 |
| Semaphore | `Semaphore` | `sem_t` | 구현 방식 차이 |
| Read-Write Lock | `SRWLock` | `pthread_rwlock_t` | 유사한 성능 |
| Event | `Event` | 없음 (조건 변수로 구현) | Windows 고유 개념 |

## Mutex 비교

### Windows: CRITICAL_SECTION (스레드 간)

`CRITICAL_SECTION`은 같은 프로세스 내 스레드 간 동기화에 최적화되어 있습니다.

```c
#include <windows.h>
#include <stdio.h>

CRITICAL_SECTION g_cs;
int g_counter = 0;

DWORD WINAPI IncrementThread(LPVOID lpParam) {
    for (int i = 0; i < 100000; i++) {
        EnterCriticalSection(&g_cs);
        g_counter++;
        LeaveCriticalSection(&g_cs);
    }
    return 0;
}

int main() {
    HANDLE threads[4];

    // Critical Section 초기화
    InitializeCriticalSection(&g_cs);

    // 스핀 카운트 설정 (성능 최적화)
    // SetCriticalSectionSpinCount(&g_cs, 2000);

    printf("Starting threads...\n");

    // 스레드 생성
    for (int i = 0; i < 4; i++) {
        threads[i] = CreateThread(NULL, 0, IncrementThread, NULL, 0, NULL);
    }

    // 모든 스레드 대기
    WaitForMultipleObjects(4, threads, TRUE, INFINITE);

    // 핸들 정리
    for (int i = 0; i < 4; i++) {
        CloseHandle(threads[i]);
    }

    printf("Final counter value: %d (expected: 400000)\n", g_counter);

    // Critical Section 정리
    DeleteCriticalSection(&g_cs);

    return 0;
}
```

### Windows: Mutex (프로세스 간)

`Mutex`는 프로세스 간 동기화도 지원하는 커널 객체입니다.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hMutex;
    int counter = 0;

    // 명명된 Mutex 생성 (프로세스 간 공유 가능)
    hMutex = CreateMutex(
        NULL,                   // 기본 보안 속성
        FALSE,                  // 초기 소유권 없음
        TEXT("Global\\MyMutex") // 이름 (NULL이면 익명)
    );

    if (hMutex == NULL) {
        printf("CreateMutex failed: %d\n", GetLastError());
        return 1;
    }

    // Mutex 획득 (최대 5초 대기)
    DWORD dwWaitResult = WaitForSingleObject(hMutex, 5000);

    switch (dwWaitResult) {
        case WAIT_OBJECT_0:
            printf("Mutex acquired\n");

            // 임계 영역
            counter++;
            Sleep(100);

            // Mutex 해제
            if (!ReleaseMutex(hMutex)) {
                printf("ReleaseMutex failed: %d\n", GetLastError());
            }
            break;

        case WAIT_TIMEOUT:
            printf("Mutex wait timeout\n");
            break;

        case WAIT_ABANDONED:
            printf("Mutex was abandoned\n");
            break;

        case WAIT_FAILED:
            printf("Wait failed: %d\n", GetLastError());
            break;
    }

    CloseHandle(hMutex);
    return 0;
}
```

### POSIX: pthread_mutex_t

```c
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
int g_counter = 0;

void* increment_thread(void* arg) {
    for (int i = 0; i < 100000; i++) {
        pthread_mutex_lock(&g_mutex);
        g_counter++;
        pthread_mutex_unlock(&g_mutex);
    }
    return NULL;
}

int main() {
    pthread_t threads[4];

    printf("Starting threads...\n");

    // 스레드 생성
    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, increment_thread, NULL);
    }

    // 모든 스레드 대기
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("Final counter value: %d (expected: 400000)\n", g_counter);

    // Mutex 정리
    pthread_mutex_destroy(&g_mutex);

    return 0;
}
```

### POSIX: Recursive Mutex

```c
#include <pthread.h>
#include <stdio.h>

pthread_mutex_t g_recursive_mutex;

void recursive_function(int depth) {
    pthread_mutex_lock(&g_recursive_mutex);

    printf("Depth: %d\n", depth);

    if (depth > 0) {
        recursive_function(depth - 1);  // 재귀적으로 락 획득
    }

    pthread_mutex_unlock(&g_recursive_mutex);
}

void* thread_func(void* arg) {
    recursive_function(3);
    return NULL;
}

int main() {
    pthread_t thread;
    pthread_mutexattr_t attr;

    // Recursive mutex 속성 설정
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);

    // Mutex 초기화
    pthread_mutex_init(&g_recursive_mutex, &attr);

    pthread_create(&thread, NULL, thread_func, NULL);
    pthread_join(thread, NULL);

    // 정리
    pthread_mutex_destroy(&g_recursive_mutex);
    pthread_mutexattr_destroy(&attr);

    return 0;
}
```

### Mutex 타임아웃 비교

#### Windows

```c
#include <windows.h>
#include <stdio.h>

CRITICAL_SECTION g_cs;

DWORD WINAPI TimeoutExample(LPVOID lpParam) {
    // CRITICAL_SECTION은 타임아웃을 직접 지원하지 않음
    // TryEnterCriticalSection 사용
    if (TryEnterCriticalSection(&g_cs)) {
        printf("Lock acquired immediately\n");
        Sleep(100);
        LeaveCriticalSection(&g_cs);
    } else {
        printf("Lock is busy, cannot acquire\n");
    }
    return 0;
}

int main() {
    InitializeCriticalSection(&g_cs);

    EnterCriticalSection(&g_cs);

    // 다른 스레드가 락을 시도
    HANDLE hThread = CreateThread(NULL, 0, TimeoutExample, NULL, 0, NULL);
    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    LeaveCriticalSection(&g_cs);
    DeleteCriticalSection(&g_cs);

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <time.h>
#include <errno.h>

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;

void* timeout_example(void* arg) {
    struct timespec ts;

    // 현재 시간 + 1초
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += 1;

    // 타임아웃과 함께 락 시도
    int result = pthread_mutex_timedlock(&g_mutex, &ts);

    if (result == 0) {
        printf("Lock acquired within timeout\n");
        pthread_mutex_unlock(&g_mutex);
    } else if (result == ETIMEDOUT) {
        printf("Lock timeout\n");
    } else {
        printf("Lock error: %d\n", result);
    }

    return NULL;
}

int main() {
    pthread_t thread;

    pthread_mutex_lock(&g_mutex);

    // 다른 스레드가 락을 시도
    pthread_create(&thread, NULL, timeout_example, NULL);
    pthread_join(thread, NULL);

    pthread_mutex_unlock(&g_mutex);
    pthread_mutex_destroy(&g_mutex);

    return 0;
}
```

## Condition Variable 비교

### Windows: CONDITION_VARIABLE

```c
#include <windows.h>
#include <stdio.h>

CRITICAL_SECTION g_cs;
CONDITION_VARIABLE g_cv;
BOOL g_ready = FALSE;
int g_data = 0;

DWORD WINAPI ProducerThread(LPVOID lpParam) {
    for (int i = 0; i < 5; i++) {
        Sleep(1000);  // 데이터 생산 시뮬레이션

        EnterCriticalSection(&g_cs);

        g_data = i;
        g_ready = TRUE;

        printf("Produced: %d\n", g_data);

        // 조건 변수 신호
        WakeConditionVariable(&g_cv);  // 하나의 대기 스레드 깨움
        // WakeAllConditionVariable(&g_cv);  // 모든 대기 스레드 깨움

        LeaveCriticalSection(&g_cs);
    }

    return 0;
}

DWORD WINAPI ConsumerThread(LPVOID lpParam) {
    int thread_id = *(int*)lpParam;

    while (1) {
        EnterCriticalSection(&g_cs);

        // 조건이 만족될 때까지 대기
        while (!g_ready) {
            // 락을 해제하고 신호 대기, 신호 받으면 락 재획득
            SleepConditionVariableCS(&g_cv, &g_cs, INFINITE);
        }

        printf("Consumer %d consumed: %d\n", thread_id, g_data);
        g_ready = FALSE;

        LeaveCriticalSection(&g_cs);

        if (g_data >= 4) break;
    }

    return 0;
}

int main() {
    HANDLE hProducer, hConsumer;
    int consumer_id = 1;

    InitializeCriticalSection(&g_cs);
    InitializeConditionVariable(&g_cv);

    hConsumer = CreateThread(NULL, 0, ConsumerThread, &consumer_id, 0, NULL);
    hProducer = CreateThread(NULL, 0, ProducerThread, NULL, 0, NULL);

    WaitForSingleObject(hProducer, INFINITE);
    WaitForSingleObject(hConsumer, INFINITE);

    CloseHandle(hProducer);
    CloseHandle(hConsumer);

    DeleteCriticalSection(&g_cs);
    // CONDITION_VARIABLE는 정리 함수가 없음

    return 0;
}
```

### POSIX: pthread_cond_t

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t g_cond = PTHREAD_COND_INITIALIZER;
bool g_ready = false;
int g_data = 0;

void* producer_thread(void* arg) {
    for (int i = 0; i < 5; i++) {
        sleep(1);  // 데이터 생산 시뮬레이션

        pthread_mutex_lock(&g_mutex);

        g_data = i;
        g_ready = true;

        printf("Produced: %d\n", g_data);

        // 조건 변수 신호
        pthread_cond_signal(&g_cond);  // 하나의 대기 스레드 깨움
        // pthread_cond_broadcast(&g_cond);  // 모든 대기 스레드 깨움

        pthread_mutex_unlock(&g_mutex);
    }

    return NULL;
}

void* consumer_thread(void* arg) {
    int thread_id = *(int*)arg;

    while (1) {
        pthread_mutex_lock(&g_mutex);

        // 조건이 만족될 때까지 대기
        while (!g_ready) {
            // 락을 해제하고 신호 대기, 신호 받으면 락 재획득
            pthread_cond_wait(&g_cond, &g_mutex);
        }

        printf("Consumer %d consumed: %d\n", thread_id, g_data);
        g_ready = false;

        int data = g_data;
        pthread_mutex_unlock(&g_mutex);

        if (data >= 4) break;
    }

    return NULL;
}

int main() {
    pthread_t producer, consumer;
    int consumer_id = 1;

    pthread_create(&consumer, NULL, consumer_thread, &consumer_id);
    pthread_create(&producer, NULL, producer_thread, NULL);

    pthread_join(producer, NULL);
    pthread_join(consumer, NULL);

    pthread_mutex_destroy(&g_mutex);
    pthread_cond_destroy(&g_cond);

    return 0;
}
```

### Condition Variable 타임아웃

#### Windows

```c
#include <windows.h>
#include <stdio.h>

CRITICAL_SECTION g_cs;
CONDITION_VARIABLE g_cv;
BOOL g_ready = FALSE;

DWORD WINAPI WaitWithTimeout(LPVOID lpParam) {
    EnterCriticalSection(&g_cs);

    printf("Waiting with 2 second timeout...\n");

    // 2초 타임아웃으로 대기
    BOOL result = SleepConditionVariableCS(&g_cv, &g_cs, 2000);

    if (result) {
        printf("Condition was signaled!\n");
    } else {
        DWORD error = GetLastError();
        if (error == ERROR_TIMEOUT) {
            printf("Wait timed out\n");
        } else {
            printf("Wait failed: %d\n", error);
        }
    }

    LeaveCriticalSection(&g_cs);
    return 0;
}

int main() {
    HANDLE hThread;

    InitializeCriticalSection(&g_cs);
    InitializeConditionVariable(&g_cv);

    hThread = CreateThread(NULL, 0, WaitWithTimeout, NULL, 0, NULL);

    // 신호를 보내지 않고 대기 (타임아웃 발생)
    WaitForSingleObject(hThread, INFINITE);

    CloseHandle(hThread);
    DeleteCriticalSection(&g_cs);

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <time.h>
#include <errno.h>
#include <stdbool.h>

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t g_cond = PTHREAD_COND_INITIALIZER;
bool g_ready = false;

void* wait_with_timeout(void* arg) {
    struct timespec ts;

    pthread_mutex_lock(&g_mutex);

    printf("Waiting with 2 second timeout...\n");

    // 현재 시간 + 2초
    clock_gettime(CLOCK_REALTIME, &ts);
    ts.tv_sec += 2;

    // 타임아웃으로 대기
    int result = pthread_cond_timedwait(&g_cond, &g_mutex, &ts);

    if (result == 0) {
        printf("Condition was signaled!\n");
    } else if (result == ETIMEDOUT) {
        printf("Wait timed out\n");
    } else {
        printf("Wait failed: %d\n", result);
    }

    pthread_mutex_unlock(&g_mutex);
    return NULL;
}

int main() {
    pthread_t thread;

    pthread_create(&thread, NULL, wait_with_timeout, NULL);

    // 신호를 보내지 않고 대기 (타임아웃 발생)
    pthread_join(thread, NULL);

    pthread_mutex_destroy(&g_mutex);
    pthread_cond_destroy(&g_cond);

    return 0;
}
```

## Windows Event를 POSIX로 구현

Windows의 Event 객체는 POSIX에 직접적인 대응물이 없습니다. 조건 변수로 구현할 수 있습니다.

### Windows: Event

```c
#include <windows.h>
#include <stdio.h>

// Manual-Reset Event
HANDLE g_manualEvent;

// Auto-Reset Event
HANDLE g_autoEvent;

DWORD WINAPI WaitingThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    printf("Thread %d waiting for manual event...\n", id);
    WaitForSingleObject(g_manualEvent, INFINITE);
    printf("Thread %d received manual event signal\n", id);

    printf("Thread %d waiting for auto event...\n", id);
    WaitForSingleObject(g_autoEvent, INFINITE);
    printf("Thread %d received auto event signal\n", id);

    return 0;
}

int main() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    // Manual-Reset Event: SetEvent 호출 시 모든 대기 스레드 깨움
    g_manualEvent = CreateEvent(NULL, TRUE, FALSE, NULL);

    // Auto-Reset Event: 하나의 스레드만 깨우고 자동으로 리셋
    g_autoEvent = CreateEvent(NULL, FALSE, FALSE, NULL);

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, WaitingThread, &ids[i], 0, NULL);
    }

    Sleep(1000);

    printf("\nSetting manual event (all threads wake up)...\n");
    SetEvent(g_manualEvent);

    Sleep(1000);

    printf("\nSetting auto event 3 times (one thread each time)...\n");
    for (int i = 0; i < 3; i++) {
        SetEvent(g_autoEvent);
        Sleep(500);
    }

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }

    CloseHandle(g_manualEvent);
    CloseHandle(g_autoEvent);

    return 0;
}
```

### POSIX: Event 구현

```c
#include <pthread.h>
#include <stdio.h>
#include <stdbool.h>
#include <unistd.h>

// Manual-Reset Event 구현
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    bool signaled;
} ManualEvent;

void manual_event_init(ManualEvent* event) {
    pthread_mutex_init(&event->mutex, NULL);
    pthread_cond_init(&event->cond, NULL);
    event->signaled = false;
}

void manual_event_set(ManualEvent* event) {
    pthread_mutex_lock(&event->mutex);
    event->signaled = true;
    pthread_cond_broadcast(&event->cond);  // 모든 스레드 깨움
    pthread_mutex_unlock(&event->mutex);
}

void manual_event_reset(ManualEvent* event) {
    pthread_mutex_lock(&event->mutex);
    event->signaled = false;
    pthread_mutex_unlock(&event->mutex);
}

void manual_event_wait(ManualEvent* event) {
    pthread_mutex_lock(&event->mutex);
    while (!event->signaled) {
        pthread_cond_wait(&event->cond, &event->mutex);
    }
    pthread_mutex_unlock(&event->mutex);
}

void manual_event_destroy(ManualEvent* event) {
    pthread_mutex_destroy(&event->mutex);
    pthread_cond_destroy(&event->cond);
}

// Auto-Reset Event 구현
typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    bool signaled;
} AutoEvent;

void auto_event_init(AutoEvent* event) {
    pthread_mutex_init(&event->mutex, NULL);
    pthread_cond_init(&event->cond, NULL);
    event->signaled = false;
}

void auto_event_set(AutoEvent* event) {
    pthread_mutex_lock(&event->mutex);
    event->signaled = true;
    pthread_cond_signal(&event->cond);  // 하나의 스레드만 깨움
    pthread_mutex_unlock(&event->mutex);
}

void auto_event_wait(AutoEvent* event) {
    pthread_mutex_lock(&event->mutex);
    while (!event->signaled) {
        pthread_cond_wait(&event->cond, &event->mutex);
    }
    event->signaled = false;  // 자동으로 리셋
    pthread_mutex_unlock(&event->mutex);
}

void auto_event_destroy(AutoEvent* event) {
    pthread_mutex_destroy(&event->mutex);
    pthread_cond_destroy(&event->cond);
}

// 사용 예제
ManualEvent g_manual_event;
AutoEvent g_auto_event;

void* waiting_thread(void* arg) {
    int id = *(int*)arg;

    printf("Thread %d waiting for manual event...\n", id);
    manual_event_wait(&g_manual_event);
    printf("Thread %d received manual event signal\n", id);

    printf("Thread %d waiting for auto event...\n", id);
    auto_event_wait(&g_auto_event);
    printf("Thread %d received auto event signal\n", id);

    return NULL;
}

int main() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    manual_event_init(&g_manual_event);
    auto_event_init(&g_auto_event);

    // 스레드 생성
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, waiting_thread, &ids[i]);
    }

    sleep(1);

    printf("\nSetting manual event (all threads wake up)...\n");
    manual_event_set(&g_manual_event);

    sleep(1);

    printf("\nSetting auto event 3 times (one thread each time)...\n");
    for (int i = 0; i < 3; i++) {
        auto_event_set(&g_auto_event);
        usleep(500000);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }

    manual_event_destroy(&g_manual_event);
    auto_event_destroy(&g_auto_event);

    return 0;
}
```

## 세마포어 비교

### Windows: Semaphore

```c
#include <windows.h>
#include <stdio.h>

#define MAX_SEM_COUNT 3

HANDLE g_semaphore;

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    printf("Thread %d waiting for semaphore...\n", id);

    // 세마포어 획득
    DWORD dwWaitResult = WaitForSingleObject(g_semaphore, INFINITE);

    if (dwWaitResult == WAIT_OBJECT_0) {
        printf("Thread %d acquired semaphore\n", id);

        // 리소스 사용
        Sleep(2000);

        printf("Thread %d releasing semaphore\n", id);

        // 세마포어 해제
        if (!ReleaseSemaphore(g_semaphore, 1, NULL)) {
            printf("ReleaseSemaphore error: %d\n", GetLastError());
        }
    }

    return 0;
}

int main() {
    HANDLE threads[5];
    int ids[5] = {1, 2, 3, 4, 5};

    // 세마포어 생성: 최대 3개 스레드 동시 접근
    g_semaphore = CreateSemaphore(
        NULL,           // 기본 보안 속성
        MAX_SEM_COUNT,  // 초기 카운트
        MAX_SEM_COUNT,  // 최대 카운트
        NULL            // 익명 세마포어
    );

    if (g_semaphore == NULL) {
        printf("CreateSemaphore error: %d\n", GetLastError());
        return 1;
    }

    // 5개 스레드 생성 (세마포어는 3개만 허용)
    for (int i = 0; i < 5; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
        Sleep(100);  // 순차적 생성
    }

    WaitForMultipleObjects(5, threads, TRUE, INFINITE);

    for (int i = 0; i < 5; i++) {
        CloseHandle(threads[i]);
    }

    CloseHandle(g_semaphore);

    return 0;
}
```

### POSIX: sem_t

```c
#include <pthread.h>
#include <semaphore.h>
#include <stdio.h>
#include <unistd.h>

#define MAX_SEM_COUNT 3

sem_t g_semaphore;

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    printf("Thread %d waiting for semaphore...\n", id);

    // 세마포어 획득
    sem_wait(&g_semaphore);

    printf("Thread %d acquired semaphore\n", id);

    // 리소스 사용
    sleep(2);

    printf("Thread %d releasing semaphore\n", id);

    // 세마포어 해제
    sem_post(&g_semaphore);

    return NULL;
}

int main() {
    pthread_t threads[5];
    int ids[5] = {1, 2, 3, 4, 5};

    // 세마포어 초기화: 최대 3개 스레드 동시 접근
    if (sem_init(&g_semaphore, 0, MAX_SEM_COUNT) == -1) {
        perror("sem_init failed");
        return 1;
    }

    // 5개 스레드 생성 (세마포어는 3개만 허용)
    for (int i = 0; i < 5; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
        usleep(100000);  // 순차적 생성
    }

    for (int i = 0; i < 5; i++) {
        pthread_join(threads[i], NULL);
    }

    sem_destroy(&g_semaphore);

    return 0;
}
```

### 명명된 세마포어 (프로세스 간)

#### Windows

```c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hSemaphore;

    // 명명된 세마포어 생성/열기
    hSemaphore = CreateSemaphore(
        NULL,
        1,                          // 초기 카운트
        1,                          // 최대 카운트
        TEXT("Global\\MySemaphore") // 이름
    );

    if (hSemaphore == NULL) {
        printf("CreateSemaphore failed: %d\n", GetLastError());
        return 1;
    }

    if (GetLastError() == ERROR_ALREADY_EXISTS) {
        printf("Semaphore already exists (opened)\n");
    } else {
        printf("Semaphore created\n");
    }

    printf("Acquiring semaphore...\n");
    WaitForSingleObject(hSemaphore, INFINITE);

    printf("Critical section (press Enter to release)\n");
    getchar();

    ReleaseSemaphore(hSemaphore, 1, NULL);
    CloseHandle(hSemaphore);

    return 0;
}
```

#### POSIX

```c
#include <fcntl.h>
#include <sys/stat.h>
#include <semaphore.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    sem_t* sem;

    // 명명된 세마포어 생성/열기
    sem = sem_open("/my_semaphore", O_CREAT, 0644, 1);

    if (sem == SEM_FAILED) {
        perror("sem_open failed");
        return 1;
    }

    printf("Acquiring semaphore...\n");
    sem_wait(sem);

    printf("Critical section (press Enter to release)\n");
    getchar();

    sem_post(sem);
    sem_close(sem);

    // 세마포어 제거 (마지막 프로세스에서)
    // sem_unlink("/my_semaphore");

    return 0;
}
```

## Read-Write Lock 비교

### Windows: SRWLock

```c
#include <windows.h>
#include <stdio.h>

SRWLOCK g_srwLock;
int g_sharedData = 0;

DWORD WINAPI ReaderThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    for (int i = 0; i < 5; i++) {
        // 읽기 락 획득
        AcquireSRWLockShared(&g_srwLock);

        printf("Reader %d: data = %d\n", id, g_sharedData);
        Sleep(100);

        // 읽기 락 해제
        ReleaseSRWLockShared(&g_srwLock);

        Sleep(50);
    }

    return 0;
}

DWORD WINAPI WriterThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    for (int i = 0; i < 3; i++) {
        // 쓰기 락 획득
        AcquireSRWLockExclusive(&g_srwLock);

        g_sharedData++;
        printf("Writer %d: updated data to %d\n", id, g_sharedData);
        Sleep(200);

        // 쓰기 락 해제
        ReleaseSRWLockExclusive(&g_srwLock);

        Sleep(300);
    }

    return 0;
}

int main() {
    HANDLE threads[5];
    int ids[5] = {1, 2, 3, 4, 5};

    // SRWLock 초기화 (정적 초기화)
    InitializeSRWLock(&g_srwLock);

    // 3개 읽기 스레드
    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, ReaderThread, &ids[i], 0, NULL);
    }

    // 2개 쓰기 스레드
    for (int i = 3; i < 5; i++) {
        threads[i] = CreateThread(NULL, 0, WriterThread, &ids[i], 0, NULL);
    }

    WaitForMultipleObjects(5, threads, TRUE, INFINITE);

    for (int i = 0; i < 5; i++) {
        CloseHandle(threads[i]);
    }

    // SRWLock은 정리 함수 불필요

    return 0;
}
```

### POSIX: pthread_rwlock_t

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

pthread_rwlock_t g_rwlock = PTHREAD_RWLOCK_INITIALIZER;
int g_shared_data = 0;

void* reader_thread(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < 5; i++) {
        // 읽기 락 획득
        pthread_rwlock_rdlock(&g_rwlock);

        printf("Reader %d: data = %d\n", id, g_shared_data);
        usleep(100000);

        // 읽기 락 해제
        pthread_rwlock_unlock(&g_rwlock);

        usleep(50000);
    }

    return NULL;
}

void* writer_thread(void* arg) {
    int id = *(int*)arg;

    for (int i = 0; i < 3; i++) {
        // 쓰기 락 획득
        pthread_rwlock_wrlock(&g_rwlock);

        g_shared_data++;
        printf("Writer %d: updated data to %d\n", id, g_shared_data);
        usleep(200000);

        // 쓰기 락 해제
        pthread_rwlock_unlock(&g_rwlock);

        usleep(300000);
    }

    return NULL;
}

int main() {
    pthread_t threads[5];
    int ids[5] = {1, 2, 3, 4, 5};

    // 3개 읽기 스레드
    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, reader_thread, &ids[i]);
    }

    // 2개 쓰기 스레드
    for (int i = 3; i < 5; i++) {
        pthread_create(&threads[i], NULL, writer_thread, &ids[i]);
    }

    for (int i = 0; i < 5; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_rwlock_destroy(&g_rwlock);

    return 0;
}
```

## 여러 객체 대기

### Windows: WaitForMultipleObjects

Windows의 가장 강력한 기능 중 하나입니다.

```c
#include <windows.h>
#include <stdio.h>

#define NUM_THREADS 4

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    printf("Thread %d working...\n", id);
    Sleep(1000 * id);
    printf("Thread %d done\n", id);

    return id * 10;
}

int main() {
    HANDLE threads[NUM_THREADS];
    int ids[NUM_THREADS] = {1, 2, 3, 4};

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
    }

    printf("Waiting for all threads...\n");

    // 모든 스레드 대기
    DWORD dwWaitResult = WaitForMultipleObjects(
        NUM_THREADS,    // 개수
        threads,        // 핸들 배열
        TRUE,           // TRUE = 모두 대기, FALSE = 하나만 대기
        INFINITE        // 타임아웃
    );

    if (dwWaitResult == WAIT_OBJECT_0) {
        printf("All threads completed\n");
    }

    // 종료 코드 확인
    for (int i = 0; i < NUM_THREADS; i++) {
        DWORD exitCode;
        GetExitCodeThread(threads[i], &exitCode);
        printf("Thread %d exit code: %lu\n", i, exitCode);
        CloseHandle(threads[i]);
    }

    printf("\n--- Waiting for any thread ---\n");

    // 다시 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        threads[i] = CreateThread(NULL, 0, WorkerThread, &ids[i], 0, NULL);
    }

    // 하나의 스레드만 완료되면 리턴
    dwWaitResult = WaitForMultipleObjects(NUM_THREADS, threads, FALSE, INFINITE);

    if (dwWaitResult >= WAIT_OBJECT_0 && dwWaitResult < WAIT_OBJECT_0 + NUM_THREADS) {
        int index = dwWaitResult - WAIT_OBJECT_0;
        printf("Thread %d completed first\n", index);
    }

    // 나머지 스레드 대기
    WaitForMultipleObjects(NUM_THREADS, threads, TRUE, INFINITE);

    for (int i = 0; i < NUM_THREADS; i++) {
        CloseHandle(threads[i]);
    }

    return 0;
}
```

### POSIX: 수동 구현

POSIX에는 `WaitForMultipleObjects` 같은 함수가 없으므로 수동으로 구현해야 합니다.

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>

#define NUM_THREADS 4

pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t g_cond = PTHREAD_COND_INITIALIZER;
int g_completed_count = 0;
bool g_completed[NUM_THREADS] = {false};

void* worker_thread(void* arg) {
    int id = *(int*)arg;

    printf("Thread %d working...\n", id);
    sleep(id);
    printf("Thread %d done\n", id);

    // 완료 표시
    pthread_mutex_lock(&g_mutex);
    g_completed[id - 1] = true;
    g_completed_count++;
    pthread_cond_broadcast(&g_cond);
    pthread_mutex_unlock(&g_mutex);

    return (void*)(long)(id * 10);
}

int main() {
    pthread_t threads[NUM_THREADS];
    int ids[NUM_THREADS] = {1, 2, 3, 4};

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    printf("Waiting for all threads...\n");

    // 모든 스레드 대기
    pthread_mutex_lock(&g_mutex);
    while (g_completed_count < NUM_THREADS) {
        pthread_cond_wait(&g_cond, &g_mutex);
    }
    pthread_mutex_unlock(&g_mutex);

    printf("All threads completed\n");

    // 종료 코드 확인
    for (int i = 0; i < NUM_THREADS; i++) {
        void* retval;
        pthread_join(threads[i], &retval);
        printf("Thread %d return value: %ld\n", i, (long)retval);
    }

    printf("\n--- Waiting for any thread ---\n");

    // 리셋
    g_completed_count = 0;
    for (int i = 0; i < NUM_THREADS; i++) {
        g_completed[i] = false;
    }

    // 다시 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_create(&threads[i], NULL, worker_thread, &ids[i]);
    }

    // 하나의 스레드만 완료되면 리턴
    pthread_mutex_lock(&g_mutex);
    while (g_completed_count == 0) {
        pthread_cond_wait(&g_cond, &g_mutex);
    }

    // 첫 번째 완료된 스레드 찾기
    for (int i = 0; i < NUM_THREADS; i++) {
        if (g_completed[i]) {
            printf("Thread %d completed first\n", i);
            break;
        }
    }
    pthread_mutex_unlock(&g_mutex);

    // 나머지 스레드 대기
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    pthread_mutex_destroy(&g_mutex);
    pthread_cond_destroy(&g_cond);

    return 0;
}
```

## 성능 비교

### 벤치마크: Mutex 성능

```c
#include <stdio.h>
#include <time.h>

#ifdef _WIN32
    #include <windows.h>
    CRITICAL_SECTION g_cs;
#else
    #include <pthread.h>
    pthread_mutex_t g_mutex = PTHREAD_MUTEX_INITIALIZER;
#endif

#define ITERATIONS 1000000

int main() {
    clock_t start, end;
    double cpu_time;

    #ifdef _WIN32
        InitializeCriticalSection(&g_cs);
    #endif

    printf("Benchmarking mutex lock/unlock performance...\n");
    printf("Iterations: %d\n\n", ITERATIONS);

    start = clock();

    for (int i = 0; i < ITERATIONS; i++) {
        #ifdef _WIN32
            EnterCriticalSection(&g_cs);
            LeaveCriticalSection(&g_cs);
        #else
            pthread_mutex_lock(&g_mutex);
            pthread_mutex_unlock(&g_mutex);
        #endif
    }

    end = clock();
    cpu_time = ((double)(end - start)) / CLOCKS_PER_SEC;

    printf("Total time: %.3f seconds\n", cpu_time);
    printf("Average time per lock/unlock: %.2f nanoseconds\n",
           (cpu_time / ITERATIONS) * 1000000000);

    #ifdef _WIN32
        DeleteCriticalSection(&g_cs);
    #else
        pthread_mutex_destroy(&g_mutex);
    #endif

    return 0;
}
```

### 일반적인 성능 특성

| 작업 | Windows (CRITICAL_SECTION) | POSIX (pthread_mutex_t) | 비고 |
|------|---------------------------|------------------------|------|
| Lock/Unlock (경합 없음) | ~20-50 ns | ~25-60 ns | 거의 동등 |
| Lock/Unlock (경합 있음) | ~1-5 μs | ~1-5 μs | 커널 전환 발생 |
| 조건 변수 대기/신호 | ~2-10 μs | ~2-10 μs | 거의 동등 |
| 세마포어 대기/해제 | ~100-500 ns | ~100-500 ns | 거의 동등 |
| Read-Write Lock (읽기) | ~30-70 ns | ~30-70 ns | 거의 동등 |

### 최적화 팁

#### Windows

1. **스핀 카운트 설정**
```c
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);
SetCriticalSectionSpinCount(&cs, 2000);  // 멀티코어 시스템에서 유용
```

2. **SRWLock 사용** (Vista 이상)
```c
// CRITICAL_SECTION보다 메모리 효율적
SRWLOCK lock = SRWLOCK_INIT;
AcquireSRWLockExclusive(&lock);
ReleaseSRWLockExclusive(&lock);
```

#### POSIX

1. **Adaptive Mutex**
```c
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ADAPTIVE_NP);  // Linux 전용

pthread_mutex_t mutex;
pthread_mutex_init(&mutex, &attr);
```

2. **읽기 선호 vs 쓰기 선호**
```c
pthread_rwlockattr_t attr;
pthread_rwlockattr_init(&attr);
pthread_rwlockattr_setkind_np(&attr, PTHREAD_RWLOCK_PREFER_WRITER_NONRECURSIVE_NP);

pthread_rwlock_t rwlock;
pthread_rwlock_init(&rwlock, &attr);
```

## 실용적 권장사항

### 1. Mutex 선택

| 사용 사례 | Windows | POSIX |
|----------|---------|-------|
| 스레드 간 동기화 | `CRITICAL_SECTION` | `pthread_mutex_t` |
| 프로세스 간 동기화 | `Mutex` | Named semaphore 또는 shared memory |
| 경량 동기화 | `SRWLock` | `pthread_mutex_t` |

### 2. Condition Variable

- 생산자-소비자 패턴: 둘 다 적합
- 타임아웃 필요: 둘 다 지원
- 브로드캐스트: 둘 다 지원

### 3. Semaphore

- 리소스 풀링: 둘 다 적합
- 프로세스 간 동기화: 명명된 세마포어 사용

### 4. Read-Write Lock

- 읽기가 많은 경우: `SRWLock` 또는 `pthread_rwlock_t`
- 읽기 선호 정책 필요: POSIX 속성 사용

### 5. 크로스 플랫폼

가능하면 C++11 표준 라이브러리 사용:
- `std::mutex`
- `std::condition_variable`
- `std::shared_mutex` (C++17)

## 요약

### Windows 장점
- `WaitForMultipleObjects`: 여러 객체 동시 대기
- Event 객체: 명확한 신호 메커니즘
- 명명된 객체: 프로세스 간 동기화 용이

### POSIX 장점
- 표준화: 이식성
- 간단한 API: 학습 곡선 낮음
- 유연한 속성: 세밀한 제어 가능

두 플랫폼 모두 효율적이고 강력한 동기화 메커니즘을 제공합니다. 선택은 주로 플랫폼 요구사항과 기존 코드베이스에 따라 결정됩니다.
