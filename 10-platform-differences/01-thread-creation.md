# 스레드 생성: Windows vs POSIX

## 목차
1. [개요](#개요)
2. [기본 API 비교](#기본-api-비교)
3. [스레드 ID와 핸들](#스레드-id와-핸들)
4. [스레드 종료와 정리](#스레드-종료와-정리)
5. [스레드 속성](#스레드-속성)
6. [Detached vs Joinable 스레드](#detached-vs-joinable-스레드)
7. [철학적 차이점](#철학적-차이점)
8. [실용적 권장사항](#실용적-권장사항)
9. [성능 고려사항](#성능-고려사항)

## 개요

Windows와 POSIX는 스레드 생성에 대해 근본적으로 다른 접근 방식을 제공합니다. Windows는 핸들 기반 API를 제공하며, POSIX는 opaque 타입과 함수 포인터를 사용합니다.

### 주요 차이점

| 항목 | Windows | POSIX |
|------|---------|-------|
| 메인 함수 | `CreateThread` | `pthread_create` |
| 스레드 식별자 | `HANDLE` (커널 객체) | `pthread_t` (opaque 타입) |
| 반환 타입 | `DWORD WINAPI` | `void*` |
| 기본 상태 | Joinable | Joinable |
| 종료 함수 | `ExitThread` | `pthread_exit` |
| 대기 함수 | `WaitForSingleObject` | `pthread_join` |

## 기본 API 비교

### Windows: CreateThread

```c
#include <windows.h>
#include <stdio.h>

// 스레드 함수 시그니처
DWORD WINAPI ThreadFunction(LPVOID lpParam) {
    int thread_id = *(int*)lpParam;
    printf("Windows Thread %d is running\n", thread_id);

    // 작업 수행
    Sleep(1000);

    printf("Windows Thread %d is exiting\n", thread_id);
    return 0;  // 종료 코드
}

int main() {
    HANDLE hThread;
    DWORD dwThreadId;
    int thread_arg = 1;

    // 스레드 생성
    hThread = CreateThread(
        NULL,                   // 보안 속성 (NULL = 기본값)
        0,                      // 스택 크기 (0 = 기본값)
        ThreadFunction,         // 스레드 함수
        &thread_arg,           // 스레드 함수 인자
        0,                      // 생성 플래그 (0 = 즉시 실행)
        &dwThreadId            // 스레드 ID 받을 변수
    );

    if (hThread == NULL) {
        printf("CreateThread failed: %d\n", GetLastError());
        return 1;
    }

    printf("Created thread with ID: %lu\n", dwThreadId);

    // 스레드 종료 대기
    WaitForSingleObject(hThread, INFINITE);

    // 핸들 정리
    CloseHandle(hThread);

    return 0;
}
```

### POSIX: pthread_create

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>

// 스레드 함수 시그니처
void* thread_function(void* arg) {
    int thread_id = *(int*)arg;
    printf("POSIX Thread %d is running\n", thread_id);

    // 작업 수행
    sleep(1);

    printf("POSIX Thread %d is exiting\n", thread_id);
    return NULL;  // 반환값 (또는 pthread_exit(NULL))
}

int main() {
    pthread_t thread;
    int thread_arg = 1;
    int result;

    // 스레드 생성
    result = pthread_create(
        &thread,            // 스레드 ID를 저장할 변수
        NULL,               // 스레드 속성 (NULL = 기본값)
        thread_function,    // 스레드 함수
        &thread_arg        // 스레드 함수 인자
    );

    if (result != 0) {
        printf("pthread_create failed: %d\n", result);
        return 1;
    }

    printf("Created thread with ID: %lu\n", (unsigned long)thread);

    // 스레드 종료 대기
    void* retval;
    pthread_join(thread, &retval);

    // POSIX에서는 명시적 정리 불필요
    return 0;
}
```

### 여러 스레드 생성 비교

#### Windows 버전

```c
#include <windows.h>
#include <stdio.h>

#define NUM_THREADS 5

typedef struct {
    int thread_id;
    char message[64];
} ThreadData;

DWORD WINAPI WorkerThread(LPVOID lpParam) {
    ThreadData* data = (ThreadData*)lpParam;

    printf("Thread %d: %s\n", data->thread_id, data->message);

    // 시뮬레이션 작업
    Sleep(100 * data->thread_id);

    printf("Thread %d completed\n", data->thread_id);

    // 동적 할당된 메모리 해제
    free(data);
    return 0;
}

int main() {
    HANDLE threads[NUM_THREADS];
    DWORD threadIds[NUM_THREADS];

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        ThreadData* data = (ThreadData*)malloc(sizeof(ThreadData));
        data->thread_id = i;
        snprintf(data->message, sizeof(data->message),
                 "Worker thread %d starting", i);

        threads[i] = CreateThread(
            NULL,
            0,
            WorkerThread,
            data,
            0,
            &threadIds[i]
        );

        if (threads[i] == NULL) {
            printf("Failed to create thread %d\n", i);
            return 1;
        }
    }

    // 모든 스레드 대기 (한 번에)
    WaitForMultipleObjects(NUM_THREADS, threads, TRUE, INFINITE);

    // 핸들 정리
    for (int i = 0; i < NUM_THREADS; i++) {
        CloseHandle(threads[i]);
    }

    printf("All threads completed\n");
    return 0;
}
```

#### POSIX 버전

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

#define NUM_THREADS 5

typedef struct {
    int thread_id;
    char message[64];
} ThreadData;

void* worker_thread(void* arg) {
    ThreadData* data = (ThreadData*)arg;

    printf("Thread %d: %s\n", data->thread_id, data->message);

    // 시뮬레이션 작업
    usleep(100000 * data->thread_id);  // 마이크로초

    printf("Thread %d completed\n", data->thread_id);

    // 동적 할당된 메모리 해제
    free(data);
    return NULL;
}

int main() {
    pthread_t threads[NUM_THREADS];
    int result;

    // 스레드 생성
    for (int i = 0; i < NUM_THREADS; i++) {
        ThreadData* data = (ThreadData*)malloc(sizeof(ThreadData));
        data->thread_id = i;
        snprintf(data->message, sizeof(data->message),
                 "Worker thread %d starting", i);

        result = pthread_create(&threads[i], NULL, worker_thread, data);

        if (result != 0) {
            printf("Failed to create thread %d: %d\n", i, result);
            return 1;
        }
    }

    // 모든 스레드 대기 (하나씩)
    for (int i = 0; i < NUM_THREADS; i++) {
        pthread_join(threads[i], NULL);
    }

    printf("All threads completed\n");
    return 0;
}
```

## 스레드 ID와 핸들

### Windows 스레드 식별

Windows에서는 두 가지 식별자를 제공합니다:

1. **HANDLE**: 커널 객체에 대한 참조, 동기화와 제어에 사용
2. **Thread ID (DWORD)**: 고유한 숫자 식별자

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI ThreadFunc(LPVOID lpParam) {
    DWORD myThreadId = GetCurrentThreadId();
    HANDLE myThreadHandle = GetCurrentThread();  // 의사 핸들

    printf("Thread ID: %lu\n", myThreadId);
    printf("Thread Handle: %p\n", myThreadHandle);

    return 0;
}

int main() {
    HANDLE hThread;
    DWORD dwThreadId;

    hThread = CreateThread(NULL, 0, ThreadFunc, NULL, 0, &dwThreadId);

    printf("Main - Created Thread ID: %lu\n", dwThreadId);
    printf("Main - Thread Handle: %p\n", hThread);

    // 중요: hThread는 실제 핸들
    // GetCurrentThread()는 의사 핸들 (중복 불가능)

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    return 0;
}
```

### POSIX 스레드 식별

POSIX에서 `pthread_t`는 opaque 타입으로 직접 비교할 수 없습니다:

```c
#include <pthread.h>
#include <stdio.h>

void* thread_func(void* arg) {
    pthread_t my_thread_id = pthread_self();

    printf("My thread ID: %lu\n", (unsigned long)my_thread_id);

    // 스레드 ID 비교
    if (pthread_equal(my_thread_id, pthread_self())) {
        printf("Thread IDs match\n");
    }

    return NULL;
}

int main() {
    pthread_t thread;

    pthread_create(&thread, NULL, thread_func, NULL);

    printf("Main - Created thread ID: %lu\n", (unsigned long)thread);
    printf("Main thread ID: %lu\n", (unsigned long)pthread_self());

    pthread_join(thread, NULL);

    return 0;
}
```

## 스레드 종료와 정리

### 명시적 종료

#### Windows

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI ExitExample(LPVOID lpParam) {
    int* value = (int*)lpParam;

    printf("Thread starting with value: %d\n", *value);

    if (*value < 0) {
        printf("Invalid value, exiting early\n");
        ExitThread(1);  // 종료 코드 1
        // 이 이후 코드는 실행되지 않음
    }

    printf("Thread continuing normally\n");
    return 0;  // 정상 종료
}

int main() {
    HANDLE hThread1, hThread2;
    int val1 = 5, val2 = -1;
    DWORD exitCode;

    hThread1 = CreateThread(NULL, 0, ExitExample, &val1, 0, NULL);
    hThread2 = CreateThread(NULL, 0, ExitExample, &val2, 0, NULL);

    WaitForSingleObject(hThread1, INFINITE);
    WaitForSingleObject(hThread2, INFINITE);

    // 종료 코드 확인
    GetExitCodeThread(hThread1, &exitCode);
    printf("Thread 1 exit code: %lu\n", exitCode);

    GetExitCodeThread(hThread2, &exitCode);
    printf("Thread 2 exit code: %lu\n", exitCode);

    CloseHandle(hThread1);
    CloseHandle(hThread2);

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void* exit_example(void* arg) {
    int* value = (int*)arg;

    printf("Thread starting with value: %d\n", *value);

    if (*value < 0) {
        printf("Invalid value, exiting early\n");
        int* ret = malloc(sizeof(int));
        *ret = 1;
        pthread_exit(ret);  // 반환값 포인터
        // 이 이후 코드는 실행되지 않음
    }

    printf("Thread continuing normally\n");
    int* ret = malloc(sizeof(int));
    *ret = 0;
    return ret;  // 정상 종료
}

int main() {
    pthread_t thread1, thread2;
    int val1 = 5, val2 = -1;
    void* retval;

    pthread_create(&thread1, NULL, exit_example, &val1);
    pthread_create(&thread2, NULL, exit_example, &val2);

    // 반환값 확인
    pthread_join(thread1, &retval);
    printf("Thread 1 return value: %d\n", *(int*)retval);
    free(retval);

    pthread_join(thread2, &retval);
    printf("Thread 2 return value: %d\n", *(int*)retval);
    free(retval);

    return 0;
}
```

### 스레드 취소 (Cancellation)

#### Windows - 직접 지원 없음

Windows는 스레드 취소를 직접 지원하지 않습니다. 대신 플래그나 이벤트를 사용합니다:

```c
#include <windows.h>
#include <stdio.h>

volatile BOOL g_shouldExit = FALSE;

DWORD WINAPI CancellableThread(LPVOID lpParam) {
    int count = 0;

    while (!g_shouldExit) {
        printf("Working... %d\n", count++);
        Sleep(500);

        // 주기적으로 종료 플래그 확인
        if (g_shouldExit) {
            printf("Cancellation requested, cleaning up...\n");
            break;
        }
    }

    printf("Thread exiting gracefully\n");
    return 0;
}

int main() {
    HANDLE hThread = CreateThread(NULL, 0, CancellableThread, NULL, 0, NULL);

    // 스레드 실행
    Sleep(2000);

    // 취소 요청
    printf("Requesting cancellation...\n");
    g_shouldExit = TRUE;

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    return 0;
}
```

#### POSIX - pthread_cancel

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void cleanup_handler(void* arg) {
    printf("Cleanup handler called: %s\n", (char*)arg);
}

void* cancellable_thread(void* arg) {
    int count = 0;

    // 취소 활성화
    pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
    pthread_setcanceltype(PTHREAD_CANCEL_DEFERRED, NULL);

    // 클린업 핸들러 등록
    pthread_cleanup_push(cleanup_handler, "Resource cleanup");

    while (1) {
        printf("Working... %d\n", count++);
        sleep(1);

        // 취소 포인트 (또는 sleep/read/write 등이 자동으로 취소 포인트)
        pthread_testcancel();
    }

    pthread_cleanup_pop(1);  // 1 = 핸들러 실행
    return NULL;
}

int main() {
    pthread_t thread;

    pthread_create(&thread, NULL, cancellable_thread, NULL);

    // 스레드 실행
    sleep(3);

    // 취소 요청
    printf("Requesting cancellation...\n");
    pthread_cancel(thread);

    void* retval;
    pthread_join(thread, &retval);

    if (retval == PTHREAD_CANCELED) {
        printf("Thread was cancelled\n");
    }

    return 0;
}
```

## 스레드 속성

### 스택 크기 설정

#### Windows

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI StackSizeThread(LPVOID lpParam) {
    // 큰 스택 배열
    char large_array[1024 * 1024];  // 1MB
    large_array[0] = 'A';

    printf("Thread with custom stack size running\n");
    printf("First byte: %c\n", large_array[0]);

    return 0;
}

int main() {
    HANDLE hThread;
    SIZE_T stackSize = 2 * 1024 * 1024;  // 2MB

    // 스택 크기 지정
    hThread = CreateThread(
        NULL,
        stackSize,          // 스택 크기 (바이트)
        StackSizeThread,
        NULL,
        0,                  // STACK_SIZE_PARAM_IS_A_RESERVATION 플래그 사용 가능
        NULL
    );

    if (hThread == NULL) {
        printf("Failed to create thread\n");
        return 1;
    }

    WaitForSingleObject(hThread, INFINITE);
    CloseHandle(hThread);

    return 0;
}
```

#### POSIX

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>

void* stack_size_thread(void* arg) {
    // 큰 스택 배열
    char large_array[1024 * 1024];  // 1MB
    large_array[0] = 'A';

    printf("Thread with custom stack size running\n");
    printf("First byte: %c\n", large_array[0]);

    // 스택 크기 확인
    pthread_attr_t attr;
    size_t stack_size;
    pthread_getattr_np(pthread_self(), &attr);
    pthread_attr_getstacksize(&attr, &stack_size);
    printf("Actual stack size: %zu bytes\n", stack_size);
    pthread_attr_destroy(&attr);

    return NULL;
}

int main() {
    pthread_t thread;
    pthread_attr_t attr;
    size_t stack_size = 2 * 1024 * 1024;  // 2MB

    // 속성 초기화
    pthread_attr_init(&attr);

    // 스택 크기 설정
    pthread_attr_setstacksize(&attr, stack_size);

    // 스택 크기 확인
    size_t actual_size;
    pthread_attr_getstacksize(&attr, &actual_size);
    printf("Requested stack size: %zu bytes\n", actual_size);

    // 스레드 생성
    if (pthread_create(&thread, &attr, stack_size_thread, NULL) != 0) {
        printf("Failed to create thread\n");
        return 1;
    }

    pthread_join(thread, NULL);

    // 속성 정리
    pthread_attr_destroy(&attr);

    return 0;
}
```

### 우선순위 설정

#### Windows

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI PriorityThread(LPVOID lpParam) {
    int priority = GetThreadPriority(GetCurrentThread());

    printf("Thread priority: %d\n", priority);
    printf("Working with priority...\n");

    // 작업 수행
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return 0;
}

int main() {
    HANDLE threads[3];

    // 낮은 우선순위
    threads[0] = CreateThread(NULL, 0, PriorityThread, NULL,
                              CREATE_SUSPENDED, NULL);
    SetThreadPriority(threads[0], THREAD_PRIORITY_LOWEST);

    // 보통 우선순위
    threads[1] = CreateThread(NULL, 0, PriorityThread, NULL,
                              CREATE_SUSPENDED, NULL);
    SetThreadPriority(threads[1], THREAD_PRIORITY_NORMAL);

    // 높은 우선순위
    threads[2] = CreateThread(NULL, 0, PriorityThread, NULL,
                              CREATE_SUSPENDED, NULL);
    SetThreadPriority(threads[2], THREAD_PRIORITY_HIGHEST);

    printf("Starting threads with different priorities...\n");

    // 모든 스레드 시작
    for (int i = 0; i < 3; i++) {
        ResumeThread(threads[i]);
    }

    // 대기 및 정리
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
#include <sched.h>

void* priority_thread(void* arg) {
    int policy;
    struct sched_param param;

    pthread_getschedparam(pthread_self(), &policy, &param);

    printf("Thread priority: %d (policy: %d)\n",
           param.sched_priority, policy);
    printf("Working with priority...\n");

    // 작업 수행
    volatile long sum = 0;
    for (long i = 0; i < 100000000; i++) {
        sum += i;
    }

    return NULL;
}

int main() {
    pthread_t threads[3];
    pthread_attr_t attr[3];
    struct sched_param param;

    // 우선순위 범위 확인
    int min_prio = sched_get_priority_min(SCHED_FIFO);
    int max_prio = sched_get_priority_max(SCHED_FIFO);
    printf("Priority range: %d - %d\n", min_prio, max_prio);

    for (int i = 0; i < 3; i++) {
        pthread_attr_init(&attr[i]);

        // 상속 정책 설정
        pthread_attr_setinheritsched(&attr[i], PTHREAD_EXPLICIT_SCHED);
        pthread_attr_setschedpolicy(&attr[i], SCHED_FIFO);

        // 우선순위 설정 (낮음, 중간, 높음)
        param.sched_priority = min_prio + (max_prio - min_prio) * i / 2;
        pthread_attr_setschedparam(&attr[i], &param);

        printf("Creating thread %d with priority %d\n",
               i, param.sched_priority);
    }

    // 스레드 생성 (root 권한 필요할 수 있음)
    for (int i = 0; i < 3; i++) {
        if (pthread_create(&threads[i], &attr[i], priority_thread, NULL) != 0) {
            printf("Failed to create thread %d (may need root privileges)\n", i);
            // 권한이 없으면 기본 속성으로 재시도
            pthread_create(&threads[i], NULL, priority_thread, NULL);
        }
    }

    // 대기 및 정리
    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
        pthread_attr_destroy(&attr[i]);
    }

    return 0;
}
```

## Detached vs Joinable 스레드

### Windows - 자동 정리

Windows에서는 핸들을 닫아도 스레드가 계속 실행됩니다:

```c
#include <windows.h>
#include <stdio.h>

DWORD WINAPI DetachedThread(LPVOID lpParam) {
    printf("Detached thread starting\n");

    for (int i = 0; i < 5; i++) {
        printf("Detached thread working... %d\n", i);
        Sleep(500);
    }

    printf("Detached thread exiting\n");
    return 0;
}

int main() {
    HANDLE hThread = CreateThread(NULL, 0, DetachedThread, NULL, 0, NULL);

    if (hThread == NULL) {
        printf("Failed to create thread\n");
        return 1;
    }

    // 핸들을 즉시 닫음 (스레드는 계속 실행)
    CloseHandle(hThread);

    printf("Main thread continues, handle closed\n");

    // 스레드가 완료될 때까지 대기
    // (핸들이 없으므로 다른 동기화 방법 필요)
    Sleep(3000);

    printf("Main thread exiting\n");
    return 0;
}
```

### POSIX - pthread_detach

```c
#include <pthread.h>
#include <stdio.h>
#include <unistd.h>

void* detached_thread(void* arg) {
    printf("Detached thread starting\n");

    for (int i = 0; i < 5; i++) {
        printf("Detached thread working... %d\n", i);
        sleep(1);
    }

    printf("Detached thread exiting\n");
    return NULL;
}

void* joinable_thread(void* arg) {
    printf("Joinable thread starting\n");
    sleep(1);
    printf("Joinable thread exiting\n");
    return NULL;
}

int main() {
    pthread_t thread1, thread2;

    // Detached 스레드 (생성 시)
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
    pthread_create(&thread1, &attr, detached_thread, NULL);
    pthread_attr_destroy(&attr);

    // Joinable 스레드를 나중에 detach
    pthread_create(&thread2, NULL, joinable_thread, NULL);
    pthread_detach(thread2);  // 명시적으로 detach

    printf("Main thread continues\n");

    // Detached 스레드는 join할 수 없음
    // pthread_join(thread1, NULL);  // 에러!
    // pthread_join(thread2, NULL);  // 에러!

    // 스레드가 완료될 시간 대기
    sleep(3);

    printf("Main thread exiting\n");
    return 0;
}
```

### Detached 스레드의 실용적 사용

#### Windows - 백그라운드 작업

```c
#include <windows.h>
#include <stdio.h>

typedef struct {
    int task_id;
    char task_name[64];
} TaskData;

DWORD WINAPI BackgroundTask(LPVOID lpParam) {
    TaskData* task = (TaskData*)lpParam;

    printf("Background task %d (%s) starting\n",
           task->task_id, task->task_name);

    // 긴 작업 수행
    Sleep(2000);

    printf("Background task %d completed\n", task->task_id);

    // 자원 정리
    free(task);
    return 0;
}

int main() {
    // 여러 백그라운드 작업 시작
    for (int i = 0; i < 3; i++) {
        TaskData* task = (TaskData*)malloc(sizeof(TaskData));
        task->task_id = i;
        snprintf(task->task_name, sizeof(task->task_name),
                 "Task_%d", i);

        HANDLE hThread = CreateThread(NULL, 0, BackgroundTask, task, 0, NULL);

        if (hThread != NULL) {
            // 핸들을 즉시 닫음 (스레드는 계속 실행)
            CloseHandle(hThread);
        }
    }

    printf("All background tasks started, main continues\n");

    // 메인 작업 수행
    Sleep(3000);

    printf("Main thread exiting\n");
    return 0;
}
```

#### POSIX - 백그라운드 작업

```c
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

typedef struct {
    int task_id;
    char task_name[64];
} TaskData;

void* background_task(void* arg) {
    TaskData* task = (TaskData*)arg;

    printf("Background task %d (%s) starting\n",
           task->task_id, task->task_name);

    // 긴 작업 수행
    sleep(2);

    printf("Background task %d completed\n", task->task_id);

    // 자원 정리
    free(task);
    return NULL;
}

int main() {
    pthread_attr_t attr;

    // Detached 속성 설정
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);

    // 여러 백그라운드 작업 시작
    for (int i = 0; i < 3; i++) {
        TaskData* task = (TaskData*)malloc(sizeof(TaskData));
        task->task_id = i;
        snprintf(task->task_name, sizeof(task->task_name),
                 "Task_%d", i);

        pthread_t thread;
        pthread_create(&thread, &attr, background_task, task);
    }

    pthread_attr_destroy(&attr);

    printf("All background tasks started, main continues\n");

    // 메인 작업 수행
    sleep(3);

    printf("Main thread exiting\n");
    return 0;
}
```

## 철학적 차이점

### Windows 접근 방식

1. **커널 객체 기반**: 스레드는 커널 객체이며 핸들을 통해 관리
2. **명시적 자원 관리**: `CloseHandle()`로 핸들을 명시적으로 닫아야 함
3. **유연한 동기화**: `WaitForMultipleObjects`로 여러 객체 대기 가능
4. **보안 속성**: 스레드 생성 시 보안 속성 지정 가능

### POSIX 접근 방식

1. **Opaque 타입**: `pthread_t`는 불투명한 타입으로 구현 숨김
2. **자동 자원 관리**: Join 또는 detach 후 자동으로 정리
3. **속성 기반**: `pthread_attr_t`로 다양한 속성을 미리 설정
4. **표준화**: 이식성을 위한 표준 API

## 실용적 권장사항

### 1. 일반적인 사용

```c
// Windows: 간단한 케이스
HANDLE hThread = CreateThread(NULL, 0, ThreadFunc, &data, 0, NULL);
WaitForSingleObject(hThread, INFINITE);
CloseHandle(hThread);

// POSIX: 간단한 케이스
pthread_t thread;
pthread_create(&thread, NULL, thread_func, &data);
pthread_join(thread, NULL);
```

### 2. 백그라운드 작업

```c
// Windows: Fire-and-forget
HANDLE hThread = CreateThread(NULL, 0, BackgroundFunc, data, 0, NULL);
CloseHandle(hThread);  // 핸들만 닫음, 스레드는 계속 실행

// POSIX: Fire-and-forget
pthread_t thread;
pthread_create(&thread, NULL, background_func, data);
pthread_detach(thread);  // 자동 정리
```

### 3. 크로스 플랫폼 래퍼

```c
// 플랫폼 독립적 인터페이스
#ifdef _WIN32
    #include <windows.h>
    typedef HANDLE thread_t;
    typedef DWORD (WINAPI *thread_func_t)(void*);

    #define THREAD_RETURN DWORD WINAPI
    #define thread_create(t, f, a) \
        (*(t) = CreateThread(NULL, 0, f, a, 0, NULL)) != NULL
    #define thread_join(t) \
        (WaitForSingleObject(t, INFINITE) == WAIT_OBJECT_0 && CloseHandle(t))
#else
    #include <pthread.h>
    typedef pthread_t thread_t;
    typedef void* (*thread_func_t)(void*);

    #define THREAD_RETURN void*
    #define thread_create(t, f, a) \
        (pthread_create(t, NULL, f, a) == 0)
    #define thread_join(t) \
        (pthread_join(t, NULL) == 0)
#endif

// 사용 예
THREAD_RETURN worker_thread(void* arg) {
    // 플랫폼 독립적 코드
    return 0;
}

int main() {
    thread_t thread;

    if (thread_create(&thread, worker_thread, NULL)) {
        thread_join(thread);
    }

    return 0;
}
```

## 성능 고려사항

### 스레드 생성 오버헤드

| 플랫폼 | 평균 생성 시간 | 특징 |
|--------|---------------|------|
| Windows | 약 50-100 μs | 커널 객체 생성 오버헤드 |
| Linux (POSIX) | 약 20-50 μs | 경량 프로세스로 구현 |

### 최적화 팁

1. **스레드 풀 사용**: 반복적인 생성/소멸 비용 절감
2. **적절한 스택 크기**: 기본값이 너무 크면 메모리 낭비
3. **우선순위 신중히 사용**: 잘못된 우선순위는 성능 저하
4. **Detached 스레드**: Join이 필요없으면 detach로 자원 절약

### 벤치마크 예제

```c
#ifdef _WIN32
    #include <windows.h>
    DWORD WINAPI empty_thread(void* arg) { return 0; }
#else
    #include <pthread.h>
    void* empty_thread(void* arg) { return NULL; }
#endif

#include <stdio.h>
#include <time.h>

int main() {
    const int NUM_ITERATIONS = 1000;
    clock_t start, end;

    start = clock();

    for (int i = 0; i < NUM_ITERATIONS; i++) {
        #ifdef _WIN32
            HANDLE hThread = CreateThread(NULL, 0, empty_thread, NULL, 0, NULL);
            WaitForSingleObject(hThread, INFINITE);
            CloseHandle(hThread);
        #else
            pthread_t thread;
            pthread_create(&thread, NULL, empty_thread, NULL);
            pthread_join(thread, NULL);
        #endif
    }

    end = clock();
    double cpu_time = ((double)(end - start)) / CLOCKS_PER_SEC;

    printf("Created and joined %d threads in %.3f seconds\n",
           NUM_ITERATIONS, cpu_time);
    printf("Average time per thread: %.2f microseconds\n",
           (cpu_time / NUM_ITERATIONS) * 1000000);

    return 0;
}
```

## 요약

| 기능 | Windows 권장 | POSIX 권장 |
|------|-------------|-----------|
| 단순 스레드 | `CreateThread` | `pthread_create` |
| 백그라운드 작업 | 핸들 즉시 닫기 | `pthread_detach` |
| 여러 스레드 대기 | `WaitForMultipleObjects` | 루프로 `pthread_join` |
| 스택 크기 설정 | `CreateThread` 두 번째 인자 | `pthread_attr_setstacksize` |
| 우선순위 설정 | `SetThreadPriority` | `pthread_attr_setschedparam` |
| 크로스 플랫폼 | C++11 `std::thread` 사용 권장 | C++11 `std::thread` 사용 권장 |

두 플랫폼 모두 강력하고 효율적인 스레드 API를 제공하지만, 철학적 차이를 이해하면 각 플랫폼에서 더 나은 코드를 작성할 수 있습니다.
