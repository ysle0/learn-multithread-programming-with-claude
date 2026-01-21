# 프로세스 간 통신 (IPC): Windows vs POSIX

## 목차
1. [개요](#개요)
2. [명명된 동기화 객체](#명명된-동기화-객체)
3. [공유 메모리](#공유-메모리)
4. [파이프와 소켓](#파이프와-소켓)
5. [메시지 큐](#메시지-큐)
6. [보안과 권한](#보안과-권한)
7. [성능 비교](#성능-비교)
8. [실용적 권장사항](#실용적-권장사항)

## 개요

프로세스 간 통신(IPC)은 독립적인 프로세스들이 데이터를 주고받는 메커니즘입니다. Windows와 POSIX는 다양한 IPC 방법을 제공합니다.

### IPC 방법 비교

| 방법 | Windows | POSIX | 용도 |
|------|---------|-------|------|
| 명명된 Mutex | `CreateMutex` | Named semaphore | 동기화 |
| 명명된 Semaphore | `CreateSemaphore` | `sem_open` | 동기화 |
| 명명된 Event | `CreateEvent` | 없음 (구현 필요) | 신호 |
| 공유 메모리 | File Mapping | `shm_open` + `mmap` | 대용량 데이터 |
| 파이프 | Named Pipe | FIFO / Unix pipe | 스트림 데이터 |
| 소켓 | Winsock | Unix Domain Socket | 범용 통신 |
| 메시지 큐 | Mailslot / Queue | `mq_open` | 메시지 전달 |

## 명명된 동기화 객체

### Windows: Named Mutex

```c
// producer.c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hMutex;

    // 명명된 뮤텍스 생성
    hMutex = CreateMutex(
        NULL,                       // 기본 보안
        FALSE,                      // 초기 소유권 없음
        TEXT("Global\\MyAppMutex")  // 이름
    );

    if (hMutex == NULL) {
        printf("CreateMutex failed: %d\n", GetLastError());
        return 1;
    }

    if (GetLastError() == ERROR_ALREADY_EXISTS) {
        printf("Mutex already exists\n");
    } else {
        printf("Mutex created\n");
    }

    printf("Press Enter to acquire mutex...\n");
    getchar();

    // Mutex 획득
    DWORD dwWaitResult = WaitForSingleObject(hMutex, 5000);

    if (dwWaitResult == WAIT_OBJECT_0) {
        printf("Mutex acquired! Holding for 10 seconds...\n");
        printf("Run another instance to test blocking\n");

        Sleep(10000);

        printf("Releasing mutex\n");
        ReleaseMutex(hMutex);
    } else if (dwWaitResult == WAIT_TIMEOUT) {
        printf("Mutex wait timed out\n");
    } else if (dwWaitResult == WAIT_ABANDONED) {
        printf("Mutex was abandoned\n");
    }

    CloseHandle(hMutex);
    return 0;
}
```

### POSIX: Named Semaphore

```c
// producer.c
#include <fcntl.h>
#include <sys/stat.h>
#include <semaphore.h>
#include <stdio.h>
#include <unistd.h>

int main() {
    sem_t* sem;

    // 명명된 세마포어 생성 (바이너리 세마포어로 뮤텍스처럼 사용)
    sem = sem_open("/myapp_mutex", O_CREAT, 0644, 1);

    if (sem == SEM_FAILED) {
        perror("sem_open failed");
        return 1;
    }

    printf("Semaphore opened\n");
    printf("Press Enter to acquire semaphore...\n");
    getchar();

    // 세마포어 획득
    printf("Acquiring semaphore...\n");
    if (sem_wait(sem) == 0) {
        printf("Semaphore acquired! Holding for 10 seconds...\n");
        printf("Run another instance to test blocking\n");

        sleep(10);

        printf("Releasing semaphore\n");
        sem_post(sem);
    } else {
        perror("sem_wait failed");
    }

    sem_close(sem);

    // 마지막 프로세스가 세마포어 제거
    // sem_unlink("/myapp_mutex");

    return 0;
}
```

### Windows: Named Event

```c
// signal_sender.c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hEvent;

    // Manual-reset event 생성
    hEvent = CreateEvent(
        NULL,                       // 기본 보안
        TRUE,                       // Manual-reset
        FALSE,                      // 초기 상태: non-signaled
        TEXT("Global\\MyAppEvent")  // 이름
    );

    if (hEvent == NULL) {
        printf("CreateEvent failed: %d\n", GetLastError());
        return 1;
    }

    printf("Event created. Press Enter to signal...\n");
    getchar();

    printf("Signaling event...\n");
    SetEvent(hEvent);

    printf("Press Enter to reset event...\n");
    getchar();

    ResetEvent(hEvent);
    printf("Event reset\n");

    CloseHandle(hEvent);
    return 0;
}

// signal_receiver.c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hEvent;

    // 기존 이벤트 열기
    hEvent = OpenEvent(
        EVENT_ALL_ACCESS,          // 접근 권한
        FALSE,                     // 상속 안 함
        TEXT("Global\\MyAppEvent") // 이름
    );

    if (hEvent == NULL) {
        printf("OpenEvent failed: %d\n", GetLastError());
        return 1;
    }

    printf("Waiting for event...\n");

    DWORD dwWaitResult = WaitForSingleObject(hEvent, INFINITE);

    if (dwWaitResult == WAIT_OBJECT_0) {
        printf("Event was signaled!\n");
    }

    CloseHandle(hEvent);
    return 0;
}
```

### POSIX: Event 구현 (공유 메모리 + Condition Variable)

```c
// event.h
#ifndef EVENT_H
#define EVENT_H

#include <pthread.h>
#include <stdbool.h>

typedef struct {
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    bool signaled;
    bool manual_reset;
} Event;

#endif

// event_impl.c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include "event.h"

// 명명된 이벤트 생성/열기
Event* event_open(const char* name, bool manual_reset) {
    char shm_name[256];
    snprintf(shm_name, sizeof(shm_name), "/event_%s", name);

    // 공유 메모리 생성/열기
    int fd = shm_open(shm_name, O_CREAT | O_RDWR, 0644);
    if (fd == -1) {
        perror("shm_open");
        return NULL;
    }

    // 크기 설정
    ftruncate(fd, sizeof(Event));

    // 매핑
    Event* event = mmap(NULL, sizeof(Event),
                        PROT_READ | PROT_WRITE,
                        MAP_SHARED, fd, 0);
    close(fd);

    if (event == MAP_FAILED) {
        perror("mmap");
        return NULL;
    }

    // 첫 생성 시 초기화
    pthread_mutexattr_t mutex_attr;
    pthread_mutexattr_init(&mutex_attr);
    pthread_mutexattr_setpshared(&mutex_attr, PTHREAD_PROCESS_SHARED);

    pthread_condattr_t cond_attr;
    pthread_condattr_init(&cond_attr);
    pthread_condattr_setpshared(&cond_attr, PTHREAD_PROCESS_SHARED);

    pthread_mutex_init(&event->mutex, &mutex_attr);
    pthread_cond_init(&event->cond, &cond_attr);
    event->signaled = false;
    event->manual_reset = manual_reset;

    pthread_mutexattr_destroy(&mutex_attr);
    pthread_condattr_destroy(&cond_attr);

    return event;
}

void event_set(Event* event) {
    pthread_mutex_lock(&event->mutex);
    event->signaled = true;
    pthread_cond_broadcast(&event->cond);
    pthread_mutex_unlock(&event->mutex);
}

void event_reset(Event* event) {
    pthread_mutex_lock(&event->mutex);
    event->signaled = false;
    pthread_mutex_unlock(&event->mutex);
}

void event_wait(Event* event) {
    pthread_mutex_lock(&event->mutex);

    while (!event->signaled) {
        pthread_cond_wait(&event->cond, &event->mutex);
    }

    // Auto-reset event
    if (!event->manual_reset) {
        event->signaled = false;
    }

    pthread_mutex_unlock(&event->mutex);
}

void event_close(Event* event) {
    munmap(event, sizeof(Event));
}

void event_unlink(const char* name) {
    char shm_name[256];
    snprintf(shm_name, sizeof(shm_name), "/event_%s", name);
    shm_unlink(shm_name);
}

// signal_sender.c
#include <stdio.h>
#include "event.h"

int main() {
    Event* event = event_open("myapp_event", true);

    printf("Event created. Press Enter to signal...\n");
    getchar();

    printf("Signaling event...\n");
    event_set(event);

    printf("Press Enter to reset event...\n");
    getchar();

    event_reset(event);
    printf("Event reset\n");

    event_close(event);
    return 0;
}

// signal_receiver.c
#include <stdio.h>
#include "event.h"

int main() {
    Event* event = event_open("myapp_event", true);

    printf("Waiting for event...\n");
    event_wait(event);

    printf("Event was signaled!\n");

    event_close(event);
    return 0;
}
```

## 공유 메모리

### Windows: File Mapping

```c
// writer.c
#include <windows.h>
#include <stdio.h>

#define SHM_SIZE 4096

typedef struct {
    int counter;
    char message[256];
} SharedData;

int main() {
    HANDLE hMapFile;
    SharedData* pSharedData;

    // 파일 매핑 객체 생성
    hMapFile = CreateFileMapping(
        INVALID_HANDLE_VALUE,      // 페이징 파일 사용
        NULL,                      // 기본 보안
        PAGE_READWRITE,            // 읽기/쓰기 권한
        0,                         // 최대 크기 (상위 32비트)
        SHM_SIZE,                  // 최대 크기 (하위 32비트)
        TEXT("Global\\MySharedMem") // 이름
    );

    if (hMapFile == NULL) {
        printf("CreateFileMapping failed: %d\n", GetLastError());
        return 1;
    }

    // 메모리 매핑
    pSharedData = (SharedData*)MapViewOfFile(
        hMapFile,
        FILE_MAP_ALL_ACCESS,
        0,
        0,
        SHM_SIZE
    );

    if (pSharedData == NULL) {
        printf("MapViewOfFile failed: %d\n", GetLastError());
        CloseHandle(hMapFile);
        return 1;
    }

    printf("Shared memory created\n");

    // 데이터 쓰기
    for (int i = 0; i < 10; i++) {
        pSharedData->counter = i;
        snprintf(pSharedData->message, sizeof(pSharedData->message),
                 "Message %d from writer", i);

        printf("Wrote: counter=%d, message='%s'\n",
               pSharedData->counter, pSharedData->message);

        Sleep(1000);
    }

    // 정리
    UnmapViewOfFile(pSharedData);
    CloseHandle(hMapFile);

    return 0;
}

// reader.c
#include <windows.h>
#include <stdio.h>

#define SHM_SIZE 4096

typedef struct {
    int counter;
    char message[256];
} SharedData;

int main() {
    HANDLE hMapFile;
    SharedData* pSharedData;

    // 기존 파일 매핑 열기
    hMapFile = OpenFileMapping(
        FILE_MAP_ALL_ACCESS,
        FALSE,
        TEXT("Global\\MySharedMem")
    );

    if (hMapFile == NULL) {
        printf("OpenFileMapping failed: %d\n", GetLastError());
        printf("Make sure writer is running\n");
        return 1;
    }

    // 메모리 매핑
    pSharedData = (SharedData*)MapViewOfFile(
        hMapFile,
        FILE_MAP_ALL_ACCESS,
        0,
        0,
        SHM_SIZE
    );

    if (pSharedData == NULL) {
        printf("MapViewOfFile failed: %d\n", GetLastError());
        CloseHandle(hMapFile);
        return 1;
    }

    printf("Connected to shared memory\n");

    // 데이터 읽기
    for (int i = 0; i < 10; i++) {
        printf("Read: counter=%d, message='%s'\n",
               pSharedData->counter, pSharedData->message);

        Sleep(1000);
    }

    // 정리
    UnmapViewOfFile(pSharedData);
    CloseHandle(hMapFile);

    return 0;
}
```

### POSIX: shm_open + mmap

```c
// writer.c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define SHM_NAME "/mysharedmem"
#define SHM_SIZE 4096

typedef struct {
    int counter;
    char message[256];
} SharedData;

int main() {
    int fd;
    SharedData* shared_data;

    // 공유 메모리 생성
    fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0644);
    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    // 크기 설정
    if (ftruncate(fd, SHM_SIZE) == -1) {
        perror("ftruncate");
        close(fd);
        return 1;
    }

    // 메모리 매핑
    shared_data = mmap(NULL, SHM_SIZE,
                       PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, 0);

    if (shared_data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    close(fd);  // 매핑 후 fd는 닫아도 됨

    printf("Shared memory created\n");

    // 데이터 쓰기
    for (int i = 0; i < 10; i++) {
        shared_data->counter = i;
        snprintf(shared_data->message, sizeof(shared_data->message),
                 "Message %d from writer", i);

        printf("Wrote: counter=%d, message='%s'\n",
               shared_data->counter, shared_data->message);

        sleep(1);
    }

    // 정리
    munmap(shared_data, SHM_SIZE);
    shm_unlink(SHM_NAME);  // 마지막 프로세스가 제거

    return 0;
}

// reader.c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>

#define SHM_NAME "/mysharedmem"
#define SHM_SIZE 4096

typedef struct {
    int counter;
    char message[256];
} SharedData;

int main() {
    int fd;
    SharedData* shared_data;

    // 기존 공유 메모리 열기
    fd = shm_open(SHM_NAME, O_RDWR, 0644);
    if (fd == -1) {
        perror("shm_open");
        printf("Make sure writer is running\n");
        return 1;
    }

    // 메모리 매핑
    shared_data = mmap(NULL, SHM_SIZE,
                       PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, 0);

    if (shared_data == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    close(fd);

    printf("Connected to shared memory\n");

    // 데이터 읽기
    for (int i = 0; i < 10; i++) {
        printf("Read: counter=%d, message='%s'\n",
               shared_data->counter, shared_data->message);

        sleep(1);
    }

    // 정리
    munmap(shared_data, SHM_SIZE);

    return 0;
}
```

### 공유 메모리 + Mutex 동기화

#### Windows

```c
// shared_counter.c
#include <windows.h>
#include <stdio.h>

#define SHM_SIZE 4096

typedef struct {
    int counter;
} SharedData;

int main() {
    HANDLE hMapFile, hMutex;
    SharedData* pSharedData;

    // 공유 메모리 생성
    hMapFile = CreateFileMapping(
        INVALID_HANDLE_VALUE,
        NULL,
        PAGE_READWRITE,
        0,
        SHM_SIZE,
        TEXT("Global\\CounterMem")
    );

    pSharedData = (SharedData*)MapViewOfFile(
        hMapFile, FILE_MAP_ALL_ACCESS, 0, 0, SHM_SIZE);

    // Mutex 생성
    hMutex = CreateMutex(NULL, FALSE, TEXT("Global\\CounterMutex"));

    printf("Process %d incrementing counter...\n", GetCurrentProcessId());

    for (int i = 0; i < 100; i++) {
        // Critical section
        WaitForSingleObject(hMutex, INFINITE);

        int old_value = pSharedData->counter;
        pSharedData->counter++;
        int new_value = pSharedData->counter;

        ReleaseMutex(hMutex);

        if (i % 10 == 0) {
            printf("Counter: %d -> %d\n", old_value, new_value);
        }

        Sleep(10);
    }

    printf("Final counter: %d\n", pSharedData->counter);

    // 정리
    UnmapViewOfFile(pSharedData);
    CloseHandle(hMapFile);
    CloseHandle(hMutex);

    return 0;
}
```

#### POSIX

```c
// shared_counter.c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <semaphore.h>
#include <unistd.h>
#include <stdio.h>

#define SHM_NAME "/counter_mem"
#define SEM_NAME "/counter_sem"

typedef struct {
    int counter;
} SharedData;

int main() {
    int fd;
    SharedData* shared_data;
    sem_t* sem;

    // 공유 메모리 생성
    fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0644);
    ftruncate(fd, sizeof(SharedData));
    shared_data = mmap(NULL, sizeof(SharedData),
                       PROT_READ | PROT_WRITE,
                       MAP_SHARED, fd, 0);
    close(fd);

    // 세마포어 생성 (뮤텍스로 사용)
    sem = sem_open(SEM_NAME, O_CREAT, 0644, 1);

    printf("Process %d incrementing counter...\n", getpid());

    for (int i = 0; i < 100; i++) {
        // Critical section
        sem_wait(sem);

        int old_value = shared_data->counter;
        shared_data->counter++;
        int new_value = shared_data->counter;

        sem_post(sem);

        if (i % 10 == 0) {
            printf("Counter: %d -> %d\n", old_value, new_value);
        }

        usleep(10000);
    }

    printf("Final counter: %d\n", shared_data->counter);

    // 정리
    munmap(shared_data, sizeof(SharedData));
    sem_close(sem);

    // 마지막 프로세스가 제거
    // shm_unlink(SHM_NAME);
    // sem_unlink(SEM_NAME);

    return 0;
}
```

## 파이프와 소켓

### Windows: Named Pipe

```c
// pipe_server.c
#include <windows.h>
#include <stdio.h>

#define PIPE_NAME TEXT("\\\\.\\pipe\\MyNamedPipe")
#define BUFFER_SIZE 512

int main() {
    HANDLE hPipe;
    char buffer[BUFFER_SIZE];
    DWORD dwRead, dwWritten;

    printf("Creating named pipe...\n");

    // Named pipe 생성
    hPipe = CreateNamedPipe(
        PIPE_NAME,                      // 파이프 이름
        PIPE_ACCESS_DUPLEX,             // 양방향
        PIPE_TYPE_MESSAGE |             // 메시지 타입
        PIPE_READMODE_MESSAGE |
        PIPE_WAIT,
        PIPE_UNLIMITED_INSTANCES,       // 무제한 인스턴스
        BUFFER_SIZE,                    // 출력 버퍼 크기
        BUFFER_SIZE,                    // 입력 버퍼 크기
        0,                              // 기본 타임아웃
        NULL                            // 기본 보안
    );

    if (hPipe == INVALID_HANDLE_VALUE) {
        printf("CreateNamedPipe failed: %d\n", GetLastError());
        return 1;
    }

    printf("Waiting for client connection...\n");

    // 클라이언트 연결 대기
    BOOL fConnected = ConnectNamedPipe(hPipe, NULL) ?
                      TRUE : (GetLastError() == ERROR_PIPE_CONNECTED);

    if (fConnected) {
        printf("Client connected\n");

        // 메시지 받기
        while (ReadFile(hPipe, buffer, BUFFER_SIZE, &dwRead, NULL)) {
            printf("Received: %s\n", buffer);

            // 응답 보내기
            char response[BUFFER_SIZE];
            snprintf(response, sizeof(response),
                     "Echo: %s", buffer);

            WriteFile(hPipe, response, strlen(response) + 1,
                     &dwWritten, NULL);

            if (strcmp(buffer, "quit") == 0) {
                break;
            }
        }

        printf("Client disconnected\n");
    }

    CloseHandle(hPipe);
    return 0;
}

// pipe_client.c
#include <windows.h>
#include <stdio.h>

#define PIPE_NAME TEXT("\\\\.\\pipe\\MyNamedPipe")
#define BUFFER_SIZE 512

int main() {
    HANDLE hPipe;
    char buffer[BUFFER_SIZE];
    DWORD dwWritten, dwRead;

    printf("Connecting to named pipe...\n");

    // 파이프 연결
    hPipe = CreateFile(
        PIPE_NAME,
        GENERIC_READ | GENERIC_WRITE,
        0,
        NULL,
        OPEN_EXISTING,
        0,
        NULL
    );

    if (hPipe == INVALID_HANDLE_VALUE) {
        printf("CreateFile failed: %d\n", GetLastError());
        printf("Make sure server is running\n");
        return 1;
    }

    printf("Connected to pipe\n");

    // 메시지 모드 설정
    DWORD dwMode = PIPE_READMODE_MESSAGE;
    SetNamedPipeHandleState(hPipe, &dwMode, NULL, NULL);

    // 메시지 주고받기
    while (1) {
        printf("Enter message (or 'quit'): ");
        fgets(buffer, BUFFER_SIZE, stdin);
        buffer[strcspn(buffer, "\n")] = 0;  // 개행 제거

        // 메시지 보내기
        WriteFile(hPipe, buffer, strlen(buffer) + 1, &dwWritten, NULL);

        // 응답 받기
        if (ReadFile(hPipe, buffer, BUFFER_SIZE, &dwRead, NULL)) {
            printf("Response: %s\n", buffer);
        }

        if (strcmp(buffer, "Echo: quit") == 0) {
            break;
        }
    }

    CloseHandle(hPipe);
    return 0;
}
```

### POSIX: FIFO (Named Pipe)

```c
// fifo_server.c
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define FIFO_REQUEST "/tmp/myfifo_request"
#define FIFO_RESPONSE "/tmp/myfifo_response"
#define BUFFER_SIZE 512

int main() {
    char buffer[BUFFER_SIZE];
    int fd_request, fd_response;

    // FIFO 생성
    mkfifo(FIFO_REQUEST, 0666);
    mkfifo(FIFO_RESPONSE, 0666);

    printf("Waiting for client...\n");

    // 요청 FIFO 열기 (읽기)
    fd_request = open(FIFO_REQUEST, O_RDONLY);
    printf("Client connected\n");

    // 응답 FIFO 열기 (쓰기)
    fd_response = open(FIFO_RESPONSE, O_WRONLY);

    // 메시지 받기
    while (1) {
        ssize_t bytes_read = read(fd_request, buffer, BUFFER_SIZE);

        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Received: %s\n", buffer);

            // 응답 보내기
            char response[BUFFER_SIZE];
            snprintf(response, sizeof(response), "Echo: %s", buffer);
            write(fd_response, response, strlen(response));

            if (strcmp(buffer, "quit") == 0) {
                break;
            }
        }
    }

    close(fd_request);
    close(fd_response);

    // FIFO 제거
    unlink(FIFO_REQUEST);
    unlink(FIFO_RESPONSE);

    return 0;
}

// fifo_client.c
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define FIFO_REQUEST "/tmp/myfifo_request"
#define FIFO_RESPONSE "/tmp/myfifo_response"
#define BUFFER_SIZE 512

int main() {
    char buffer[BUFFER_SIZE];
    int fd_request, fd_response;

    printf("Connecting to server...\n");

    // 요청 FIFO 열기 (쓰기)
    fd_request = open(FIFO_REQUEST, O_WRONLY);

    // 응답 FIFO 열기 (읽기)
    fd_response = open(FIFO_RESPONSE, O_RDONLY);

    printf("Connected\n");

    // 메시지 주고받기
    while (1) {
        printf("Enter message (or 'quit'): ");
        fgets(buffer, BUFFER_SIZE, stdin);
        buffer[strcspn(buffer, "\n")] = 0;

        // 메시지 보내기
        write(fd_request, buffer, strlen(buffer));

        // 응답 받기
        ssize_t bytes_read = read(fd_response, buffer, BUFFER_SIZE);
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Response: %s\n", buffer);
        }

        if (strncmp(buffer, "Echo: quit", 10) == 0) {
            break;
        }
    }

    close(fd_request);
    close(fd_response);

    return 0;
}
```

### POSIX: Unix Domain Socket

```c
// socket_server.c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define SOCKET_PATH "/tmp/mysocket"
#define BUFFER_SIZE 512

int main() {
    int server_fd, client_fd;
    struct sockaddr_un server_addr, client_addr;
    socklen_t client_len;
    char buffer[BUFFER_SIZE];

    // 소켓 생성
    server_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (server_fd == -1) {
        perror("socket");
        return 1;
    }

    // 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sun_family = AF_UNIX;
    strncpy(server_addr.sun_path, SOCKET_PATH,
            sizeof(server_addr.sun_path) - 1);

    // 기존 소켓 파일 제거
    unlink(SOCKET_PATH);

    // 바인드
    if (bind(server_fd, (struct sockaddr*)&server_addr,
             sizeof(server_addr)) == -1) {
        perror("bind");
        close(server_fd);
        return 1;
    }

    // 리슨
    if (listen(server_fd, 5) == -1) {
        perror("listen");
        close(server_fd);
        return 1;
    }

    printf("Waiting for client connection...\n");

    // 연결 수락
    client_len = sizeof(client_addr);
    client_fd = accept(server_fd, (struct sockaddr*)&client_addr,
                      &client_len);

    if (client_fd == -1) {
        perror("accept");
        close(server_fd);
        return 1;
    }

    printf("Client connected\n");

    // 메시지 받기
    while (1) {
        ssize_t bytes_read = read(client_fd, buffer, BUFFER_SIZE);

        if (bytes_read <= 0) {
            break;
        }

        buffer[bytes_read] = '\0';
        printf("Received: %s\n", buffer);

        // 응답 보내기
        char response[BUFFER_SIZE];
        snprintf(response, sizeof(response), "Echo: %s", buffer);
        write(client_fd, response, strlen(response));

        if (strcmp(buffer, "quit") == 0) {
            break;
        }
    }

    printf("Client disconnected\n");

    close(client_fd);
    close(server_fd);
    unlink(SOCKET_PATH);

    return 0;
}

// socket_client.c
#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

#define SOCKET_PATH "/tmp/mysocket"
#define BUFFER_SIZE 512

int main() {
    int client_fd;
    struct sockaddr_un server_addr;
    char buffer[BUFFER_SIZE];

    // 소켓 생성
    client_fd = socket(AF_UNIX, SOCK_STREAM, 0);
    if (client_fd == -1) {
        perror("socket");
        return 1;
    }

    // 주소 설정
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sun_family = AF_UNIX;
    strncpy(server_addr.sun_path, SOCKET_PATH,
            sizeof(server_addr.sun_path) - 1);

    // 연결
    if (connect(client_fd, (struct sockaddr*)&server_addr,
                sizeof(server_addr)) == -1) {
        perror("connect");
        printf("Make sure server is running\n");
        close(client_fd);
        return 1;
    }

    printf("Connected to server\n");

    // 메시지 주고받기
    while (1) {
        printf("Enter message (or 'quit'): ");
        fgets(buffer, BUFFER_SIZE, stdin);
        buffer[strcspn(buffer, "\n")] = 0;

        // 메시지 보내기
        write(client_fd, buffer, strlen(buffer));

        // 응답 받기
        ssize_t bytes_read = read(client_fd, buffer, BUFFER_SIZE);
        if (bytes_read > 0) {
            buffer[bytes_read] = '\0';
            printf("Response: %s\n", buffer);
        }

        if (strcmp(buffer, "Echo: quit") == 0) {
            break;
        }
    }

    close(client_fd);
    return 0;
}
```

## 메시지 큐

### POSIX: Message Queue

```c
// mq_sender.c
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>

#define MQ_NAME "/mymessagequeue"
#define MAX_MSG_SIZE 256
#define MAX_MESSAGES 10

int main() {
    mqd_t mq;
    struct mq_attr attr;
    char buffer[MAX_MSG_SIZE];

    // 메시지 큐 속성 설정
    attr.mq_flags = 0;
    attr.mq_maxmsg = MAX_MESSAGES;
    attr.mq_msgsize = MAX_MSG_SIZE;
    attr.mq_curmsgs = 0;

    // 메시지 큐 생성
    mq = mq_open(MQ_NAME, O_CREAT | O_WRONLY, 0644, &attr);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        return 1;
    }

    printf("Message queue created\n");

    // 메시지 보내기
    for (int i = 0; i < 5; i++) {
        snprintf(buffer, sizeof(buffer), "Message %d", i);

        if (mq_send(mq, buffer, strlen(buffer) + 1, 0) == -1) {
            perror("mq_send");
        } else {
            printf("Sent: %s\n", buffer);
        }

        sleep(1);
    }

    mq_close(mq);

    return 0;
}

// mq_receiver.c
#include <fcntl.h>
#include <sys/stat.h>
#include <mqueue.h>
#include <stdio.h>
#include <string.h>

#define MQ_NAME "/mymessagequeue"
#define MAX_MSG_SIZE 256

int main() {
    mqd_t mq;
    struct mq_attr attr;
    char buffer[MAX_MSG_SIZE];
    unsigned int prio;

    // 메시지 큐 열기
    mq = mq_open(MQ_NAME, O_RDONLY);
    if (mq == (mqd_t)-1) {
        perror("mq_open");
        printf("Make sure sender has created the queue\n");
        return 1;
    }

    // 속성 가져오기
    mq_getattr(mq, &attr);
    printf("Message queue opened (max size: %ld)\n", attr.mq_msgsize);

    // 메시지 받기
    while (1) {
        ssize_t bytes_read = mq_receive(mq, buffer, attr.mq_msgsize, &prio);

        if (bytes_read == -1) {
            perror("mq_receive");
            break;
        }

        printf("Received (prio %u): %s\n", prio, buffer);

        // 큐가 비었는지 확인
        mq_getattr(mq, &attr);
        if (attr.mq_curmsgs == 0) {
            printf("Queue is empty, waiting...\n");
        }
    }

    mq_close(mq);
    mq_unlink(MQ_NAME);

    return 0;
}
```

## 보안과 권한

### Windows: Security Descriptors

```c
#include <windows.h>
#include <aclapi.h>
#include <stdio.h>

int main() {
    HANDLE hMutex;
    SECURITY_ATTRIBUTES sa;
    SECURITY_DESCRIPTOR sd;

    // Security Descriptor 초기화
    InitializeSecurityDescriptor(&sd, SECURITY_DESCRIPTOR_REVISION);

    // Everyone에게 모든 권한 부여 (예제용)
    SetSecurityDescriptorDacl(&sd, TRUE, NULL, FALSE);

    sa.nLength = sizeof(SECURITY_ATTRIBUTES);
    sa.lpSecurityDescriptor = &sd;
    sa.bInheritHandle = FALSE;

    // 보안 속성과 함께 Mutex 생성
    hMutex = CreateMutex(&sa, FALSE, TEXT("Global\\SecureMutex"));

    if (hMutex == NULL) {
        printf("CreateMutex failed: %d\n", GetLastError());
        return 1;
    }

    printf("Secure mutex created\n");

    // 사용...

    CloseHandle(hMutex);
    return 0;
}
```

### POSIX: File Permissions

```c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <stdio.h>

int main() {
    int fd;

    // 특정 권한으로 공유 메모리 생성
    // 0640 = 소유자: rw-, 그룹: r--, 기타: ---
    fd = shm_open("/secure_shm", O_CREAT | O_RDWR, 0640);

    if (fd == -1) {
        perror("shm_open");
        return 1;
    }

    printf("Secure shared memory created\n");

    // 권한 확인
    struct stat st;
    fstat(fd, &st);
    printf("Permissions: %o\n", st.st_mode & 0777);

    close(fd);
    shm_unlink("/secure_shm");

    return 0;
}
```

## 성능 비교

| IPC 방법 | 처리량 | 지연시간 | 사용 사례 |
|---------|--------|---------|----------|
| 공유 메모리 | 매우 높음 | 매우 낮음 | 대용량 데이터, 고성능 |
| Unix Domain Socket | 높음 | 낮음 | 범용 통신 |
| Named Pipe | 중간 | 중간 | 스트림 데이터 |
| Message Queue | 중간 | 중간 | 메시지 기반 통신 |
| FIFO | 낮음 | 높음 | 단방향 통신 |

## 실용적 권장사항

### 1. 동기화만 필요할 때
- Windows: `CreateMutex`, `CreateSemaphore`, `CreateEvent`
- POSIX: `sem_open`

### 2. 대용량 데이터 공유
- Windows: File Mapping
- POSIX: `shm_open` + `mmap`

### 3. 스트림 통신
- Windows: Named Pipe
- POSIX: Unix Domain Socket

### 4. 메시지 기반 통신
- Windows: Mailslot (단방향) 또는 Message Queue
- POSIX: `mq_open`

### 5. 크로스 플랫폼
- TCP/IP 소켓 사용 (이식성 최고)
- 또는 추상화 라이브러리 (Boost.Interprocess 등)

## 요약

Windows와 POSIX는 다양한 IPC 메커니즘을 제공합니다. Windows는 더 통합된 API를 제공하는 반면, POSIX는 Unix 철학에 따라 작고 조합 가능한 도구들을 제공합니다. 적절한 IPC 방법을 선택하면 프로세스 간 효율적인 통신을 구현할 수 있습니다.
