# 에러 처리: Windows vs POSIX

## 목차
1. [개요](#개요)
2. [에러 코드 메커니즘](#에러-코드-메커니즘)
3. [스레드 안전한 에러 처리](#스레드-안전한-에러-처리)
4. [에러 메시지 포매팅](#에러-메시지-포매팅)
5. [에러 코드 매핑](#에러-코드-매핑)
6. [예외와 에러 처리](#예외와-에러-처리)
7. [디버깅 지원](#디버깅-지원)
8. [실용적 권장사항](#실용적-권장사항)

## 개요

Windows와 POSIX는 에러 처리에 대해 근본적으로 다른 접근 방식을 사용합니다. 이러한 차이를 이해하면 크로스 플랫폼 코드를 작성할 때 일관된 에러 처리를 구현할 수 있습니다.

### 주요 차이점

| 항목 | Windows | POSIX |
|------|---------|-------|
| 에러 저장 | `GetLastError()` | `errno` (TLS 변수) |
| 에러 설정 | `SetLastError()` | 함수 반환값 + `errno` |
| 에러 타입 | `DWORD` (32비트) | `int` |
| 스레드 안전성 | 스레드별 저장 | 스레드별 저장 |
| 에러 메시지 | `FormatMessage()` | `strerror()` |
| 에러 코드 범위 | 0 ~ 4294967295 | 1 ~ 수백 |

## 에러 코드 메커니즘

### Windows: GetLastError와 SetLastError

```c
#include <windows.h>
#include <stdio.h>

void DemonstrateWindowsErrors() {
    HANDLE hFile;
    DWORD dwError;

    printf("=== Windows Error Handling ===\n\n");

    // 1. 성공하는 작업
    hFile = CreateFile(
        TEXT("existing_file.txt"),
        GENERIC_READ,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile != INVALID_HANDLE_VALUE) {
        printf("File opened successfully\n");
        CloseHandle(hFile);
    } else {
        dwError = GetLastError();
        printf("Failed to open file, error: %lu\n", dwError);
    }

    // 2. 실패하는 작업
    hFile = CreateFile(
        TEXT("nonexistent_file.txt"),
        GENERIC_READ,
        FILE_SHARE_READ,
        NULL,
        OPEN_EXISTING,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        dwError = GetLastError();
        printf("\nExpected failure:\n");
        printf("  Error code: %lu\n", dwError);
        printf("  Error name: ");

        switch (dwError) {
            case ERROR_FILE_NOT_FOUND:
                printf("ERROR_FILE_NOT_FOUND\n");
                break;
            case ERROR_PATH_NOT_FOUND:
                printf("ERROR_PATH_NOT_FOUND\n");
                break;
            case ERROR_ACCESS_DENIED:
                printf("ERROR_ACCESS_DENIED\n");
                break;
            default:
                printf("Unknown error\n");
        }
    }

    // 3. 에러 코드 수동 설정
    SetLastError(ERROR_SUCCESS);
    printf("\nError cleared: %lu\n", GetLastError());

    SetLastError(ERROR_INVALID_PARAMETER);
    printf("Error set to ERROR_INVALID_PARAMETER: %lu\n", GetLastError());
}

// 스레드별 에러 코드 테스트
DWORD WINAPI ErrorThread(LPVOID lpParam) {
    int id = *(int*)lpParam;

    // 각 스레드는 독립적인 에러 코드를 가짐
    SetLastError(100 + id);

    Sleep(100);

    DWORD error = GetLastError();
    printf("Thread %d error: %lu\n", id, error);

    return 0;
}

void TestThreadSafeErrors() {
    HANDLE threads[3];
    int ids[3] = {1, 2, 3};

    printf("\n=== Thread-Safe Error Test ===\n");

    for (int i = 0; i < 3; i++) {
        threads[i] = CreateThread(NULL, 0, ErrorThread, &ids[i], 0, NULL);
    }

    WaitForMultipleObjects(3, threads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(threads[i]);
    }
}

int main() {
    DemonstrateWindowsErrors();
    TestThreadSafeErrors();

    return 0;
}
```

### POSIX: errno

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <pthread.h>

void demonstrate_posix_errors() {
    int fd;

    printf("=== POSIX Error Handling ===\n\n");

    // 1. 성공하는 작업
    fd = open("existing_file.txt", O_RDONLY);

    if (fd != -1) {
        printf("File opened successfully\n");
        close(fd);
    } else {
        printf("Failed to open file, errno: %d\n", errno);
    }

    // 2. 실패하는 작업
    fd = open("nonexistent_file.txt", O_RDONLY);

    if (fd == -1) {
        printf("\nExpected failure:\n");
        printf("  Error code: %d\n", errno);
        printf("  Error name: ");

        switch (errno) {
            case ENOENT:
                printf("ENOENT (No such file or directory)\n");
                break;
            case EACCES:
                printf("EACCES (Permission denied)\n");
                break;
            case EINVAL:
                printf("EINVAL (Invalid argument)\n");
                break;
            default:
                printf("Unknown error\n");
        }

        printf("  Error message: %s\n", strerror(errno));
    }

    // 3. errno 수동 설정
    errno = 0;
    printf("\nError cleared: %d\n", errno);

    errno = EINVAL;
    printf("Error set to EINVAL: %d (%s)\n", errno, strerror(errno));
}

// 스레드별 errno 테스트
void* error_thread(void* arg) {
    int id = *(int*)arg;

    // 각 스레드는 독립적인 errno를 가짐
    errno = 100 + id;

    usleep(100000);

    printf("Thread %d errno: %d\n", id, errno);

    return NULL;
}

void test_thread_safe_errors() {
    pthread_t threads[3];
    int ids[3] = {1, 2, 3};

    printf("\n=== Thread-Safe Error Test ===\n");

    for (int i = 0; i < 3; i++) {
        pthread_create(&threads[i], NULL, error_thread, &ids[i]);
    }

    for (int i = 0; i < 3; i++) {
        pthread_join(threads[i], NULL);
    }
}

int main() {
    demonstrate_posix_errors();
    test_thread_safe_errors();

    return 0;
}
```

### 에러 확인 패턴

#### Windows: 반환값과 GetLastError

```c
#include <windows.h>
#include <stdio.h>

// 패턴 1: HANDLE 반환 (INVALID_HANDLE_VALUE 또는 NULL)
void HandleReturnPattern() {
    HANDLE hFile = CreateFile(
        TEXT("test.txt"),
        GENERIC_READ,
        0,
        NULL,
        OPEN_EXISTING,
        0,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        DWORD error = GetLastError();
        printf("CreateFile failed: %lu\n", error);
        return;
    }

    // 사용...
    CloseHandle(hFile);
}

// 패턴 2: BOOL 반환 (TRUE/FALSE)
void BoolReturnPattern() {
    BOOL result = SetCurrentDirectory(TEXT("nonexistent"));

    if (!result) {
        DWORD error = GetLastError();
        printf("SetCurrentDirectory failed: %lu\n", error);
        return;
    }

    printf("Directory changed successfully\n");
}

// 패턴 3: DWORD 반환 (0 = 성공, 다른 값 = 에러 코드)
void DwordReturnPattern() {
    HANDLE hThread;
    DWORD dwThreadId;

    hThread = CreateThread(NULL, 0, NULL, NULL, 0, &dwThreadId);

    if (hThread == NULL) {
        DWORD error = GetLastError();
        printf("CreateThread failed: %lu\n", error);
        return;
    }

    // 대기...
    DWORD waitResult = WaitForSingleObject(hThread, 1000);

    switch (waitResult) {
        case WAIT_OBJECT_0:
            printf("Thread signaled\n");
            break;
        case WAIT_TIMEOUT:
            printf("Wait timed out\n");
            break;
        case WAIT_FAILED:
            printf("Wait failed: %lu\n", GetLastError());
            break;
    }

    CloseHandle(hThread);
}

int main() {
    printf("=== Windows Error Checking Patterns ===\n\n");

    printf("Pattern 1: HANDLE return\n");
    HandleReturnPattern();

    printf("\nPattern 2: BOOL return\n");
    BoolReturnPattern();

    printf("\nPattern 3: DWORD return\n");
    DwordReturnPattern();

    return 0;
}
```

#### POSIX: 반환값과 errno

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <pthread.h>

// 패턴 1: -1 반환 (실패 시, errno 설정)
void MinusOnePattern() {
    int fd = open("nonexistent.txt", O_RDONLY);

    if (fd == -1) {
        printf("open failed: %s\n", strerror(errno));
        return;
    }

    // 사용...
    close(fd);
}

// 패턴 2: 0이 아닌 값 반환 (실패 시, 반환값이 errno)
void NonZeroPattern() {
    pthread_t thread;
    int result = pthread_create(&thread, NULL, NULL, NULL);

    if (result != 0) {
        printf("pthread_create failed: %s\n", strerror(result));
        // 주의: errno를 설정하지 않음!
        return;
    }

    pthread_join(thread, NULL);
}

// 패턴 3: NULL 반환 (실패 시, errno 설정)
void NullPattern() {
    FILE* file = fopen("nonexistent.txt", "r");

    if (file == NULL) {
        printf("fopen failed: %s\n", strerror(errno));
        return;
    }

    // 사용...
    fclose(file);
}

// 패턴 4: 특별한 값 반환
void SpecialValuePattern() {
    // getpid는 항상 성공 (에러 없음)
    pid_t pid = getpid();
    printf("Process ID: %d\n", pid);

    // read는 0 (EOF), -1 (에러), 양수 (읽은 바이트 수)
    char buffer[100];
    ssize_t bytes_read = read(STDIN_FILENO, buffer, sizeof(buffer));

    if (bytes_read == -1) {
        printf("read failed: %s\n", strerror(errno));
    } else if (bytes_read == 0) {
        printf("EOF reached\n");
    } else {
        printf("Read %zd bytes\n", bytes_read);
    }
}

int main() {
    printf("=== POSIX Error Checking Patterns ===\n\n");

    printf("Pattern 1: -1 return with errno\n");
    MinusOnePattern();

    printf("\nPattern 2: Non-zero return (error code)\n");
    NonZeroPattern();

    printf("\nPattern 3: NULL return with errno\n");
    NullPattern();

    printf("\nPattern 4: Special values\n");
    SpecialValuePattern();

    return 0;
}
```

## 스레드 안전한 에러 처리

### errno의 스레드 안전성

```c
#include <pthread.h>
#include <errno.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

void* thread_errno_test(void* arg) {
    int id = *(int*)arg;

    // 실패하는 시스템 콜 (errno 설정)
    int fd = open("/nonexistent/path/file.txt", O_RDONLY);

    if (fd == -1) {
        // 각 스레드는 독립적인 errno를 가짐
        int saved_errno = errno;
        printf("Thread %d: errno = %d (%s)\n",
               id, saved_errno, strerror(saved_errno));
    }

    return NULL;
}

int main() {
    pthread_t threads[5];
    int ids[5] = {1, 2, 3, 4, 5};

    printf("=== errno Thread Safety Test ===\n");
    printf("Each thread should see the same error (ENOENT)\n\n");

    for (int i = 0; i < 5; i++) {
        pthread_create(&threads[i], NULL, thread_errno_test, &ids[i]);
    }

    for (int i = 0; i < 5; i++) {
        pthread_join(threads[i], NULL);
    }

    return 0;
}
```

### 에러 코드 저장 베스트 프랙티스

```c
#include <stdio.h>
#include <errno.h>
#include <string.h>

// 나쁜 예: errno가 덮어써질 수 있음
void bad_error_handling() {
    int fd = open("file.txt", O_RDONLY);

    if (fd == -1) {
        printf("Debug info...\n");  // 이 함수가 errno를 변경할 수 있음!
        printf("Error: %s\n", strerror(errno));  // 잘못된 에러 출력 가능
    }
}

// 좋은 예: errno를 즉시 저장
void good_error_handling() {
    int fd = open("file.txt", O_RDONLY);

    if (fd == -1) {
        int saved_errno = errno;  // 즉시 저장
        printf("Debug info...\n");
        printf("Error: %s\n", strerror(saved_errno));  // 올바른 에러 출력
    }
}

// 더 좋은 예: 에러 처리 함수
void report_error(const char* operation, int error_code) {
    fprintf(stderr, "%s failed: %s (errno: %d)\n",
            operation, strerror(error_code), error_code);
}

void better_error_handling() {
    int fd = open("file.txt", O_RDONLY);

    if (fd == -1) {
        report_error("open", errno);
        return;
    }

    // 사용...
    close(fd);
}
```

## 에러 메시지 포매팅

### Windows: FormatMessage

```c
#include <windows.h>
#include <stdio.h>

void PrintWindowsError(DWORD errorCode, const char* operation) {
    LPVOID lpMsgBuf;
    DWORD bufLen;

    bufLen = FormatMessage(
        FORMAT_MESSAGE_ALLOCATE_BUFFER |
        FORMAT_MESSAGE_FROM_SYSTEM |
        FORMAT_MESSAGE_IGNORE_INSERTS,
        NULL,
        errorCode,
        MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
        (LPTSTR)&lpMsgBuf,
        0,
        NULL
    );

    if (bufLen) {
        LPCTSTR lpMsgStr = (LPCTSTR)lpMsgBuf;
        printf("%s failed with error %lu: %s",
               operation, errorCode, lpMsgStr);

        LocalFree(lpMsgBuf);
    } else {
        printf("%s failed with error %lu (no message available)\n",
               operation, errorCode);
    }
}

void DemonstrateFormatMessage() {
    HANDLE hFile;

    printf("=== FormatMessage Examples ===\n\n");

    // 예제 1: 파일 열기 실패
    hFile = CreateFile(
        TEXT("C:\\nonexistent\\path\\file.txt"),
        GENERIC_READ,
        0,
        NULL,
        OPEN_EXISTING,
        0,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        PrintWindowsError(GetLastError(), "CreateFile");
    }

    // 예제 2: 디렉토리 생성 실패
    if (!CreateDirectory(TEXT("C:\\Windows\\testdir"), NULL)) {
        PrintWindowsError(GetLastError(), "CreateDirectory");
    }

    // 예제 3: 일반적인 에러 코드들
    printf("\n=== Common Error Messages ===\n");

    DWORD commonErrors[] = {
        ERROR_SUCCESS,
        ERROR_FILE_NOT_FOUND,
        ERROR_PATH_NOT_FOUND,
        ERROR_ACCESS_DENIED,
        ERROR_INVALID_HANDLE,
        ERROR_NOT_ENOUGH_MEMORY,
        ERROR_INVALID_PARAMETER
    };

    for (int i = 0; i < sizeof(commonErrors) / sizeof(DWORD); i++) {
        printf("\nError %lu:\n  ", commonErrors[i]);
        PrintWindowsError(commonErrors[i], "Operation");
    }
}

int main() {
    DemonstrateFormatMessage();
    return 0;
}
```

### POSIX: strerror와 perror

```c
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>

void demonstrate_strerror() {
    printf("=== strerror and perror Examples ===\n\n");

    // 예제 1: strerror 사용
    int fd = open("/nonexistent/path/file.txt", O_RDONLY);

    if (fd == -1) {
        int saved_errno = errno;
        printf("open failed:\n");
        printf("  errno: %d\n", saved_errno);
        printf("  strerror: %s\n", strerror(saved_errno));
    }

    // 예제 2: perror 사용 (stderr로 출력)
    printf("\nUsing perror:\n");
    fd = open("/nonexistent/path/file.txt", O_RDONLY);

    if (fd == -1) {
        perror("  open");  // "open: No such file or directory" 형식
    }

    // 예제 3: 스레드 안전한 strerror_r
    #ifdef _GNU_SOURCE
    printf("\nUsing strerror_r (thread-safe):\n");
    fd = open("/nonexistent/path/file.txt", O_RDONLY);

    if (fd == -1) {
        char buffer[256];
        char* result = strerror_r(errno, buffer, sizeof(buffer));
        printf("  strerror_r: %s\n", result);
    }
    #endif

    // 예제 4: 일반적인 에러 코드들
    printf("\n=== Common Error Messages ===\n");

    int commonErrors[] = {
        0,          // Success
        EPERM,      // Operation not permitted
        ENOENT,     // No such file or directory
        ESRCH,      // No such process
        EINTR,      // Interrupted system call
        EIO,        // I/O error
        ENXIO,      // No such device or address
        E2BIG,      // Argument list too long
        EBADF,      // Bad file descriptor
        EAGAIN,     // Try again
        ENOMEM,     // Out of memory
        EACCES,     // Permission denied
        EFAULT,     // Bad address
        EBUSY,      // Device or resource busy
        EEXIST,     // File exists
        EINVAL,     // Invalid argument
        EMFILE,     // Too many open files
        ENOSPC,     // No space left on device
    };

    for (int i = 0; i < sizeof(commonErrors) / sizeof(int); i++) {
        printf("\nerrno %d (%s):\n",
               commonErrors[i],
               strerror(commonErrors[i]));
    }
}

int main() {
    demonstrate_strerror();
    return 0;
}
```

### 커스텀 에러 메시지 포매팅

```c
#include <stdio.h>
#include <stdarg.h>
#include <string.h>
#include <errno.h>
#include <time.h>

// 크로스 플랫폼 에러 로깅
void log_error(const char* file, int line, const char* func,
               const char* fmt, ...) {
    char timestamp[64];
    time_t now = time(NULL);
    struct tm* tm_info = localtime(&now);

    strftime(timestamp, sizeof(timestamp), "%Y-%m-%d %H:%M:%S", tm_info);

    fprintf(stderr, "[ERROR] %s | %s:%d | %s | ",
            timestamp, file, line, func);

    va_list args;
    va_start(args, fmt);
    vfprintf(stderr, fmt, args);
    va_end(args);

    fprintf(stderr, "\n");
}

#define LOG_ERROR(...) \
    log_error(__FILE__, __LINE__, __func__, __VA_ARGS__)

// POSIX 에러와 함께 로깅
void log_errno_error(const char* file, int line, const char* func,
                     int error_code, const char* operation) {
    log_error(file, line, func, "%s failed: %s (errno: %d)",
              operation, strerror(error_code), error_code);
}

#define LOG_ERRNO_ERROR(op) \
    log_errno_error(__FILE__, __LINE__, __func__, errno, op)

// 사용 예제
void example_usage() {
    int fd = open("/nonexistent/file.txt", O_RDONLY);

    if (fd == -1) {
        LOG_ERRNO_ERROR("open");
        return;
    }

    // 작업 수행...

    if (close(fd) == -1) {
        LOG_ERRNO_ERROR("close");
    }
}

int main() {
    printf("=== Custom Error Logging Example ===\n\n");

    LOG_ERROR("This is a custom error message");
    LOG_ERROR("Error with parameter: %d", 42);

    example_usage();

    return 0;
}
```

## 에러 코드 매핑

### Windows와 POSIX 에러 코드 변환

```c
#include <stdio.h>

#ifdef _WIN32
    #include <windows.h>

    // Windows 에러를 POSIX 스타일로 변환
    int windows_error_to_errno(DWORD win_error) {
        switch (win_error) {
            case ERROR_SUCCESS:
                return 0;
            case ERROR_FILE_NOT_FOUND:
            case ERROR_PATH_NOT_FOUND:
                return ENOENT;
            case ERROR_ACCESS_DENIED:
                return EACCES;
            case ERROR_INVALID_HANDLE:
                return EBADF;
            case ERROR_NOT_ENOUGH_MEMORY:
            case ERROR_OUTOFMEMORY:
                return ENOMEM;
            case ERROR_INVALID_PARAMETER:
                return EINVAL;
            case ERROR_ALREADY_EXISTS:
                return EEXIST;
            case ERROR_TOO_MANY_OPEN_FILES:
                return EMFILE;
            case ERROR_DISK_FULL:
                return ENOSPC;
            case ERROR_BROKEN_PIPE:
                return EPIPE;
            case ERROR_TIMEOUT:
                return ETIMEDOUT;
            default:
                return EIO;  // 일반적인 I/O 에러
        }
    }

    void demonstrate_error_mapping() {
        printf("=== Windows to POSIX Error Mapping ===\n\n");

        DWORD win_errors[] = {
            ERROR_FILE_NOT_FOUND,
            ERROR_ACCESS_DENIED,
            ERROR_NOT_ENOUGH_MEMORY,
            ERROR_INVALID_PARAMETER,
            ERROR_ALREADY_EXISTS
        };

        for (int i = 0; i < sizeof(win_errors) / sizeof(DWORD); i++) {
            int posix_err = windows_error_to_errno(win_errors[i]);

            printf("Windows error %lu -> POSIX errno %d (%s)\n",
                   win_errors[i], posix_err, strerror(posix_err));
        }
    }
#else
    #include <errno.h>
    #include <string.h>

    // POSIX 에러를 Windows 스타일로 변환 (개념적)
    unsigned long errno_to_windows_error(int posix_error) {
        switch (posix_error) {
            case 0:
                return 0;  // ERROR_SUCCESS
            case ENOENT:
                return 2;  // ERROR_FILE_NOT_FOUND
            case EACCES:
                return 5;  // ERROR_ACCESS_DENIED
            case EBADF:
                return 6;  // ERROR_INVALID_HANDLE
            case ENOMEM:
                return 8;  // ERROR_NOT_ENOUGH_MEMORY
            case EINVAL:
                return 87; // ERROR_INVALID_PARAMETER
            case EEXIST:
                return 80; // ERROR_ALREADY_EXISTS
            case EMFILE:
                return 4;  // ERROR_TOO_MANY_OPEN_FILES
            case ENOSPC:
                return 112; // ERROR_DISK_FULL
            case EPIPE:
                return 109; // ERROR_BROKEN_PIPE
            case ETIMEDOUT:
                return 1460; // ERROR_TIMEOUT
            default:
                return 1;  // ERROR_INVALID_FUNCTION
        }
    }

    void demonstrate_error_mapping() {
        printf("=== POSIX to Windows Error Mapping ===\n\n");

        int posix_errors[] = {
            ENOENT,
            EACCES,
            ENOMEM,
            EINVAL,
            EEXIST
        };

        for (int i = 0; i < sizeof(posix_errors) / sizeof(int); i++) {
            unsigned long win_err = errno_to_windows_error(posix_errors[i]);

            printf("POSIX errno %d (%s) -> Windows error %lu\n",
                   posix_errors[i], strerror(posix_errors[i]), win_err);
        }
    }
#endif

int main() {
    demonstrate_error_mapping();
    return 0;
}
```

## 예외와 에러 처리

### C++에서의 크로스 플랫폼 에러 처리

```cpp
#include <iostream>
#include <system_error>
#include <string>

#ifdef _WIN32
    #include <windows.h>
#else
    #include <errno.h>
    #include <string.h>
#endif

// 크로스 플랫폼 시스템 에러 예외
class SystemError : public std::system_error {
public:
    #ifdef _WIN32
    SystemError(const std::string& operation)
        : std::system_error(GetLastError(), std::system_category(),
                           operation) {}

    SystemError(DWORD error_code, const std::string& operation)
        : std::system_error(error_code, std::system_category(),
                           operation) {}
    #else
    SystemError(const std::string& operation)
        : std::system_error(errno, std::system_category(),
                           operation) {}

    SystemError(int error_code, const std::string& operation)
        : std::system_error(error_code, std::system_category(),
                           operation) {}
    #endif
};

// 사용 예제
void riskyOperation() {
    #ifdef _WIN32
    HANDLE hFile = CreateFileA(
        "nonexistent.txt",
        GENERIC_READ,
        0,
        NULL,
        OPEN_EXISTING,
        0,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        throw SystemError("CreateFile");
    }

    CloseHandle(hFile);
    #else
    int fd = open("nonexistent.txt", O_RDONLY);

    if (fd == -1) {
        throw SystemError("open");
    }

    close(fd);
    #endif
}

int main() {
    std::cout << "=== C++ Exception-Based Error Handling ===\n\n";

    try {
        riskyOperation();
    } catch (const SystemError& e) {
        std::cerr << "System error: " << e.what() << "\n";
        std::cerr << "Error code: " << e.code() << "\n";
    } catch (const std::exception& e) {
        std::cerr << "Exception: " << e.what() << "\n";
    }

    return 0;
}
```

## 디버깅 지원

### Windows: DebugBreak와 OutputDebugString

```c
#include <windows.h>
#include <stdio.h>

void DebugLog(const char* format, ...) {
    char buffer[1024];
    va_list args;

    va_start(args, format);
    vsnprintf(buffer, sizeof(buffer), format, args);
    va_end(args);

    // 디버거 출력
    OutputDebugStringA(buffer);

    // 콘솔 출력
    printf("%s", buffer);
}

void DemonstrateWindowsDebugging() {
    printf("=== Windows Debugging Support ===\n\n");

    DebugLog("This message appears in debugger output\n");

    // 디버거가 연결되어 있는지 확인
    if (IsDebuggerPresent()) {
        DebugLog("Debugger is attached\n");

        // 조건부 브레이크포인트
        int value = 42;
        if (value > 40) {
            DebugLog("Breaking into debugger...\n");
            // DebugBreak();  // 주석 해제하면 디버거에서 중단
        }
    } else {
        DebugLog("No debugger attached\n");
    }

    // 에러 발생 시 자동 브레이크
    HANDLE hFile = CreateFile(
        TEXT("nonexistent.txt"),
        GENERIC_READ,
        0,
        NULL,
        OPEN_EXISTING,
        0,
        NULL
    );

    if (hFile == INVALID_HANDLE_VALUE) {
        DWORD error = GetLastError();
        DebugLog("CreateFile failed: %lu\n", error);

        if (IsDebuggerPresent()) {
            // DebugBreak();  // 에러 발생 시 중단
        }
    }
}

int main() {
    DemonstrateWindowsDebugging();
    return 0;
}
```

### POSIX: assert와 디버깅

```c
#include <stdio.h>
#include <assert.h>
#include <signal.h>
#include <unistd.h>
#include <sys/types.h>

void debug_log(const char* format, ...) {
    va_list args;

    va_start(args, format);
    vfprintf(stderr, format, args);
    va_end(args);
}

// 커스텀 assertion
#ifdef NDEBUG
    #define DEBUG_ASSERT(expr) ((void)0)
#else
    #define DEBUG_ASSERT(expr) \
        do { \
            if (!(expr)) { \
                fprintf(stderr, "Assertion failed: %s\n", #expr); \
                fprintf(stderr, "  File: %s\n", __FILE__); \
                fprintf(stderr, "  Line: %d\n", __LINE__); \
                fprintf(stderr, "  Function: %s\n", __func__); \
                raise(SIGTRAP);  /* 디버거에서 중단 */ \
            } \
        } while(0)
#endif

void demonstrate_posix_debugging() {
    printf("=== POSIX Debugging Support ===\n\n");

    debug_log("This is a debug message\n");

    // gdb가 attach되어 있는지 확인
    char buf[256];
    snprintf(buf, sizeof(buf), "/proc/%d/status", getpid());

    FILE* f = fopen(buf, "r");
    if (f) {
        while (fgets(buf, sizeof(buf), f)) {
            if (strncmp(buf, "TracerPid:", 10) == 0) {
                int tracer_pid = atoi(buf + 10);
                if (tracer_pid != 0) {
                    debug_log("Debugger is attached (PID: %d)\n", tracer_pid);
                } else {
                    debug_log("No debugger attached\n");
                }
                break;
            }
        }
        fclose(f);
    }

    // Assertion 예제
    int value = 42;
    DEBUG_ASSERT(value > 0);
    DEBUG_ASSERT(value < 100);

    // 실패하는 assertion (주석 처리됨)
    // DEBUG_ASSERT(value > 50);  // 이것은 중단됨
}

int main() {
    demonstrate_posix_debugging();
    return 0;
}
```

## 실용적 권장사항

### 1. 일관된 에러 처리 패턴

```c
// error_utils.h
#ifndef ERROR_UTILS_H
#define ERROR_UTILS_H

#ifdef _WIN32
    #include <windows.h>
    typedef DWORD error_code_t;
    #define get_last_error() GetLastError()
    #define set_last_error(code) SetLastError(code)
#else
    #include <errno.h>
    typedef int error_code_t;
    #define get_last_error() errno
    #define set_last_error(code) (errno = (code))
#endif

// 에러 메시지 가져오기
const char* get_error_message(error_code_t error_code);

// 에러 로깅
void log_error_ex(const char* file, int line, const char* func,
                  error_code_t error_code, const char* operation);

#define log_error(op) \
    log_error_ex(__FILE__, __LINE__, __func__, get_last_error(), op)

#endif // ERROR_UTILS_H
```

### 2. 에러 복구 전략

```c
#include <stdio.h>

typedef enum {
    RETRY_NONE,
    RETRY_IMMEDIATE,
    RETRY_WITH_DELAY,
    RETRY_WITH_BACKOFF
} RetryStrategy;

int perform_operation_with_retry(
    int (*operation)(void*),
    void* context,
    RetryStrategy strategy,
    int max_retries
) {
    int attempts = 0;
    int delay_ms = 100;

    while (attempts < max_retries) {
        int result = operation(context);

        if (result == 0) {
            return 0;  // 성공
        }

        attempts++;

        if (attempts >= max_retries) {
            break;
        }

        // 재시도 전략에 따라 대기
        switch (strategy) {
            case RETRY_IMMEDIATE:
                break;

            case RETRY_WITH_DELAY:
                #ifdef _WIN32
                Sleep(delay_ms);
                #else
                usleep(delay_ms * 1000);
                #endif
                break;

            case RETRY_WITH_BACKOFF:
                #ifdef _WIN32
                Sleep(delay_ms);
                #else
                usleep(delay_ms * 1000);
                #endif
                delay_ms *= 2;  // 지수 백오프
                break;

            default:
                return -1;
        }

        printf("Retrying operation (attempt %d/%d)...\n",
               attempts + 1, max_retries);
    }

    return -1;  // 실패
}
```

### 3. 크로스 플랫폼 에러 처리 래퍼

```c
// xplatform_error.h
#ifndef XPLATFORM_ERROR_H
#define XPLATFORM_ERROR_H

typedef struct {
    int code;
    char message[256];
} ErrorInfo;

// 마지막 에러 정보 가져오기
void get_error_info(ErrorInfo* info);

// 에러 메시지 출력
void print_error(const char* operation, const ErrorInfo* info);

#endif

// xplatform_error.c
#include "xplatform_error.h"
#include <stdio.h>
#include <string.h>

#ifdef _WIN32
    #include <windows.h>

    void get_error_info(ErrorInfo* info) {
        DWORD error_code = GetLastError();
        info->code = (int)error_code;

        FormatMessageA(
            FORMAT_MESSAGE_FROM_SYSTEM |
            FORMAT_MESSAGE_IGNORE_INSERTS,
            NULL,
            error_code,
            MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT),
            info->message,
            sizeof(info->message),
            NULL
        );
    }
#else
    #include <errno.h>

    void get_error_info(ErrorInfo* info) {
        info->code = errno;
        strncpy(info->message, strerror(errno), sizeof(info->message) - 1);
        info->message[sizeof(info->message) - 1] = '\0';
    }
#endif

void print_error(const char* operation, const ErrorInfo* info) {
    fprintf(stderr, "%s failed (code %d): %s\n",
            operation, info->code, info->message);
}
```

## 요약

### 에러 처리 비교

| 항목 | Windows | POSIX |
|------|---------|-------|
| 에러 가져오기 | `GetLastError()` | `errno` 또는 반환값 |
| 에러 설정 | `SetLastError()` | `errno = value` |
| 스레드 안전성 | 예 | 예 |
| 에러 메시지 | `FormatMessage()` | `strerror()`, `perror()` |
| 에러 타입 | `DWORD` (0-4B) | `int` (1-100+) |

### 베스트 프랙티스

1. **즉시 에러 저장**: 다른 함수 호출 전에 에러 코드 저장
2. **일관된 패턴**: 모든 함수에서 동일한 에러 처리 패턴 사용
3. **의미 있는 메시지**: 컨텍스트 정보를 포함한 에러 메시지
4. **에러 복구**: 가능한 경우 재시도 또는 폴백 제공
5. **크로스 플랫폼**: 추상화 레이어로 플랫폼 차이 숨김

적절한 에러 처리는 견고한 애플리케이션의 핵심입니다. Windows와 POSIX의 차이를 이해하고 일관된 추상화를 사용하면 유지보수하기 쉬운 크로스 플랫폼 코드를 작성할 수 있습니다.
