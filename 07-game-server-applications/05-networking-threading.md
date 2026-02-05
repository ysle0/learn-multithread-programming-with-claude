# 네트워크 I/O와 Threading

## 개요

네트워크 I/O는 게임 서버에서 병목이 되는 경우가 많습니다. 수천 개의 동시 연결을 효율적으로 처리하려면 고급 I/O 모델과 threading 전략이 필요합니다. 이 문서에서는 비동기 I/O, I/O completion ports (IOCP), epoll, 네트워크 처리를 위한 thread pool, 그리고 zero-copy 기법을 다룹니다.

## 목차

1. [I/O 모델](#io-models)
2. [플랫폼별 API](#platform-specific-apis)
3. [Threading 전략](#threading-strategies)
4. [프로토콜 설계](#protocol-design)
5. [성능 최적화](#performance-optimization)
6. [실제 구현 사례](#real-world-implementations)

## I/O 모델

### 1. Blocking I/O

**개념:** I/O 작업이 완료될 때까지 thread가 차단됩니다.

```
Thread Timeline (Blocking I/O)
─────────────────────────────────────
Thread 1: [recv]─────────[recv]─────
           ↓ blocks      ↓ blocks
          waiting       waiting

장점: 단순함
단점: 연결당 하나의 thread 필요, 확장성 부족
```

**구현:**

```cpp
// Blocking I/O - 단순하지만 확장성이 부족함
void HandleClient(int socket_fd) {
    char buffer[4096];

    while (true) {
        // 데이터가 도착할 때까지 차단
        ssize_t bytes_read = recv(socket_fd, buffer, sizeof(buffer), 0);

        if (bytes_read <= 0) {
            break; // 연결 종료 또는 오류
        }

        // 데이터 처리
        ProcessData(buffer, bytes_read);

        // 응답 전송 (역시 차단됨)
        send(socket_fd, response, response_length, 0);
    }

    close(socket_fd);
}

// 연결당 thread 방식 (약 1000개 이상의 연결에서는 확장 불가)
void RunServer() {
    int listen_fd = CreateListenSocket(8080);

    while (true) {
        int client_fd = accept(listen_fd, nullptr, nullptr);

        // 각 연결마다 thread 생성
        std::thread client_thread(HandleClient, client_fd);
        client_thread.detach();
    }
}
```

### 2. Non-blocking I/O (select/poll 사용)

**개념:** 단일 thread가 여러 소켓을 모니터링합니다.

```
Event Loop (select/poll)
──────────────────────────────────────
         ┌─────────────┐
         │  select()   │───────┐
         └─────────────┘       │
                │              │
         준비된 소켓들          │
                │              │
         ┌──────▼──────┐       │
         │  I/O 처리   │       │
         └─────────────┘       │
                │              │
                └──────────────┘

장점: 여러 연결 처리 가능
단점: FD 스캔에 O(n), 10K+ 연결에서 확장 불가
```

**구현:**

```cpp
// select를 사용한 Non-blocking I/O
class SelectServer {
public:
    void Run(uint16_t port) {
        int listen_fd = CreateListenSocket(port);
        SetNonBlocking(listen_fd);

        fd_set master_set;
        FD_ZERO(&master_set);
        FD_SET(listen_fd, &master_set);

        int max_fd = listen_fd;

        while (true) {
            fd_set read_set = master_set;

            // 소켓에서 활동이 발생할 때까지 대기
            int activity = select(max_fd + 1, &read_set, nullptr, nullptr, nullptr);

            if (activity < 0) {
                perror("select");
                break;
            }

            // 각 소켓 확인
            for (int fd = 0; fd <= max_fd; ++fd) {
                if (!FD_ISSET(fd, &read_set)) {
                    continue;
                }

                if (fd == listen_fd) {
                    // 새 연결
                    int client_fd = accept(listen_fd, nullptr, nullptr);
                    SetNonBlocking(client_fd);
                    FD_SET(client_fd, &master_set);
                    max_fd = std::max(max_fd, client_fd);
                } else {
                    // 기존 연결에서 데이터 수신
                    char buffer[4096];
                    ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                    if (bytes <= 0) {
                        // 연결 종료
                        close(fd);
                        FD_CLR(fd, &master_set);
                    } else {
                        ProcessData(fd, buffer, bytes);
                    }
                }
            }
        }
    }

private:
    void SetNonBlocking(int fd) {
        int flags = fcntl(fd, F_GETFL, 0);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    }
};
```

### 3. 비동기 I/O (Reactor 패턴)

**개념:** 콜백을 사용하는 이벤트 기반 I/O입니다.

```
Reactor Pattern
─────────────────────────────────────
    ┌────────────────────┐
    │   Event Demux      │
    │  (epoll/kqueue)    │
    └──────────┬─────────┘
               │
        ┌──────┴───────┐
        │              │
        ▼              ▼
   ┌─────────┐   ┌─────────┐
   │Handler 1│   │Handler 2│
   │(socket A)│   │(socket B)│
   └─────────┘   └─────────┘

장점: 확장 가능, 효율적
단점: 콜백 복잡성
```

### 4. Proactor 패턴 (비동기 완료)

**개념:** I/O 작업이 비동기적으로 완료되며, 완료 시 콜백이 호출됩니다.

```
Proactor Pattern (IOCP)
─────────────────────────────────────
  I/O 시작
       │
       ▼
  ┌─────────────┐
  │   Kernel    │
  │  (Async I/O)│
  └──────┬──────┘
         │
      완료
         │
         ▼
  ┌─────────────┐
  │   완료      │
  │   Handler   │
  └─────────────┘

장점: 진정한 비동기, 확장 가능
단점: 복잡함, 플랫폼 의존적
```

## 플랫폼별 API

### Linux: epoll

**특징:**
- Edge-triggered와 level-triggered 모드 지원
- 준비된 이벤트에 대해 O(1) 성능
- 100K+ 연결까지 확장 가능

**구현:**

```cpp
// epoll 기반 서버 (Linux)
class EpollServer {
public:
    EpollServer() {
        epoll_fd_ = epoll_create1(0);
        if (epoll_fd_ < 0) {
            throw std::runtime_error("epoll_create1 failed");
        }
    }

    ~EpollServer() {
        if (epoll_fd_ >= 0) {
            close(epoll_fd_);
        }
    }

    void Run(uint16_t port) {
        int listen_fd = CreateListenSocket(port);
        SetNonBlocking(listen_fd);

        // 리슨 소켓을 epoll에 추가
        AddSocket(listen_fd, EPOLLIN | EPOLLET); // Edge-triggered

        constexpr int MAX_EVENTS = 1024;
        struct epoll_event events[MAX_EVENTS];

        while (running_) {
            // 이벤트 대기
            int num_events = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].data.fd;

                if (fd == listen_fd) {
                    // 새 연결 수락
                    AcceptConnections(listen_fd);
                } else if (events[i].events & EPOLLIN) {
                    // 읽을 데이터 준비됨
                    HandleRead(fd);
                } else if (events[i].events & EPOLLOUT) {
                    // 쓰기 준비됨
                    HandleWrite(fd);
                } else if (events[i].events & (EPOLLHUP | EPOLLERR)) {
                    // 연결 오류 또는 종료
                    HandleDisconnect(fd);
                }
            }
        }
    }

private:
    void AddSocket(int fd, uint32_t events) {
        struct epoll_event ev;
        ev.events = events;
        ev.data.fd = fd;

        if (epoll_ctl(epoll_fd_, EPOLL_CTL_ADD, fd, &ev) < 0) {
            perror("epoll_ctl");
        }
    }

    void AcceptConnections(int listen_fd) {
        while (true) {
            int client_fd = accept(listen_fd, nullptr, nullptr);

            if (client_fd < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // 더 이상 연결 없음
                    break;
                } else {
                    perror("accept");
                    break;
                }
            }

            SetNonBlocking(client_fd);
            AddSocket(client_fd, EPOLLIN | EPOLLOUT | EPOLLET);

            // 연결 상태 생성
            connections_[client_fd] = std::make_unique<Connection>(client_fd);
        }
    }

    void HandleRead(int fd) {
        auto it = connections_.find(fd);
        if (it == connections_.end()) return;

        auto& conn = it->second;

        while (true) {
            char buffer[4096];
            ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

            if (bytes < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // 더 이상 데이터 없음
                    break;
                } else {
                    // 오류
                    HandleDisconnect(fd);
                    return;
                }
            } else if (bytes == 0) {
                // 연결 종료
                HandleDisconnect(fd);
                return;
            }

            // 데이터 처리
            conn->input_buffer.append(buffer, bytes);
        }

        // 완전한 패킷 처리
        ProcessPackets(conn.get());
    }

    void HandleWrite(int fd) {
        auto it = connections_.find(fd);
        if (it == connections_.end()) return;

        auto& conn = it->second;

        while (!conn->output_buffer.empty()) {
            ssize_t bytes = send(fd,
                conn->output_buffer.data(),
                conn->output_buffer.size(),
                0);

            if (bytes < 0) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    // 차단될 수 있음, 나중에 재시도
                    break;
                } else {
                    // 오류
                    HandleDisconnect(fd);
                    return;
                }
            }

            // 전송된 데이터를 버퍼에서 제거
            conn->output_buffer.erase(0, bytes);
        }
    }

    void HandleDisconnect(int fd) {
        epoll_ctl(epoll_fd_, EPOLL_CTL_DEL, fd, nullptr);
        close(fd);
        connections_.erase(fd);
    }

    void SetNonBlocking(int fd) {
        int flags = fcntl(fd, F_GETFL, 0);
        fcntl(fd, F_SETFL, flags | O_NONBLOCK);
    }

private:
    int epoll_fd_;
    std::atomic<bool> running_{true};

    struct Connection {
        int fd;
        std::string input_buffer;
        std::string output_buffer;

        explicit Connection(int fd) : fd(fd) {}
    };

    std::unordered_map<int, std::unique_ptr<Connection>> connections_;
};
```

### Windows: I/O Completion Ports (IOCP)

**특징:**
- 완료 기반 (Proactor 패턴)
- 100K+ 연결까지 확장 가능
- Thread pool과 통합

**구현:**

```cpp
// IOCP 기반 서버 (Windows)
#ifdef _WIN32

class IOCPServer {
public:
    IOCPServer(int num_threads = 0) {
        // I/O completion port 생성
        iocp_handle_ = CreateIoCompletionPort(
            INVALID_HANDLE_VALUE,
            nullptr,
            0,
            num_threads // 0 = 프로세서 수
        );

        if (!iocp_handle_) {
            throw std::runtime_error("CreateIoCompletionPort failed");
        }
    }

    ~IOCPServer() {
        if (iocp_handle_) {
            CloseHandle(iocp_handle_);
        }
    }

    void Run(uint16_t port) {
        // 리슨 소켓 생성
        listen_socket_ = CreateListenSocket(port);

        // 리슨 소켓을 IOCP에 연결
        CreateIoCompletionPort(
            (HANDLE)listen_socket_,
            iocp_handle_,
            (ULONG_PTR)nullptr,
            0
        );

        // 워커 thread 시작
        int num_threads = std::thread::hardware_concurrency();
        for (int i = 0; i < num_threads; ++i) {
            worker_threads_.emplace_back([this]() {
                WorkerThread();
            });
        }

        // 연결 수락
        AcceptLoop();
    }

private:
    struct IOContext {
        OVERLAPPED overlapped;
        enum class Operation {
            Accept,
            Receive,
            Send
        } operation;

        WSABUF wsa_buf;
        char buffer[4096];
        SOCKET socket;
    };

    struct Connection {
        SOCKET socket;
        std::mutex send_mutex;
        std::queue<std::vector<char>> send_queue;
    };

    void WorkerThread() {
        while (true) {
            DWORD bytes_transferred;
            ULONG_PTR completion_key;
            LPOVERLAPPED overlapped;

            // I/O 완료 대기
            BOOL result = GetQueuedCompletionStatus(
                iocp_handle_,
                &bytes_transferred,
                &completion_key,
                &overlapped,
                INFINITE
            );

            if (!overlapped) {
                // 종료 신호
                break;
            }

            auto* io_context = CONTAINING_RECORD(
                overlapped,
                IOContext,
                overlapped
            );

            auto* connection = reinterpret_cast<Connection*>(completion_key);

            if (!result || bytes_transferred == 0) {
                // 연결 종료 또는 오류
                HandleDisconnect(connection, io_context);
                continue;
            }

            // I/O 완료 처리
            switch (io_context->operation) {
                case IOContext::Operation::Accept:
                    HandleAcceptComplete(io_context, bytes_transferred);
                    break;

                case IOContext::Operation::Receive:
                    HandleReceiveComplete(connection, io_context, bytes_transferred);
                    break;

                case IOContext::Operation::Send:
                    HandleSendComplete(connection, io_context, bytes_transferred);
                    break;
            }
        }
    }

    void AcceptLoop() {
        while (running_) {
            // Accept 작업 등록
            auto* io_context = new IOContext;
            ZeroMemory(&io_context->overlapped, sizeof(OVERLAPPED));
            io_context->operation = IOContext::Operation::Accept;

            io_context->socket = WSASocket(
                AF_INET,
                SOCK_STREAM,
                IPPROTO_TCP,
                nullptr,
                0,
                WSA_FLAG_OVERLAPPED
            );

            DWORD bytes_received;
            BOOL result = AcceptEx(
                listen_socket_,
                io_context->socket,
                io_context->buffer,
                0, // 초기 수신 없음
                sizeof(sockaddr_in) + 16,
                sizeof(sockaddr_in) + 16,
                &bytes_received,
                &io_context->overlapped
            );

            if (!result && WSAGetLastError() != ERROR_IO_PENDING) {
                // 오류
                closesocket(io_context->socket);
                delete io_context;
            }

            // 다음 accept 전에 잠시 대기
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }

    void HandleAcceptComplete(IOContext* io_context, DWORD bytes_transferred) {
        // 연결 생성
        auto connection = std::make_unique<Connection>();
        connection->socket = io_context->socket;

        // 소켓을 IOCP에 연결
        CreateIoCompletionPort(
            (HANDLE)connection->socket,
            iocp_handle_,
            (ULONG_PTR)connection.get(),
            0
        );

        connections_[connection->socket] = std::move(connection);

        // 초기 수신 등록
        PostReceive(connections_[io_context->socket].get());

        delete io_context;
    }

    void PostReceive(Connection* connection) {
        auto* io_context = new IOContext;
        ZeroMemory(&io_context->overlapped, sizeof(OVERLAPPED));
        io_context->operation = IOContext::Operation::Receive;
        io_context->socket = connection->socket;
        io_context->wsa_buf.buf = io_context->buffer;
        io_context->wsa_buf.len = sizeof(io_context->buffer);

        DWORD flags = 0;
        DWORD bytes_received;

        int result = WSARecv(
            connection->socket,
            &io_context->wsa_buf,
            1,
            &bytes_received,
            &flags,
            &io_context->overlapped,
            nullptr
        );

        if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
            // 오류
            delete io_context;
        }
    }

    void HandleReceiveComplete(Connection* connection, IOContext* io_context,
                               DWORD bytes_transferred) {
        // 수신된 데이터 처리
        ProcessData(connection, io_context->buffer, bytes_transferred);

        // 다음 수신 등록
        PostReceive(connection);

        delete io_context;
    }

    void PostSend(Connection* connection, const char* data, size_t length) {
        auto* io_context = new IOContext;
        ZeroMemory(&io_context->overlapped, sizeof(OVERLAPPED));
        io_context->operation = IOContext::Operation::Send;
        io_context->socket = connection->socket;

        memcpy(io_context->buffer, data, length);
        io_context->wsa_buf.buf = io_context->buffer;
        io_context->wsa_buf.len = length;

        DWORD bytes_sent;

        int result = WSASend(
            connection->socket,
            &io_context->wsa_buf,
            1,
            &bytes_sent,
            0,
            &io_context->overlapped,
            nullptr
        );

        if (result == SOCKET_ERROR && WSAGetLastError() != WSA_IO_PENDING) {
            // 오류
            delete io_context;
        }
    }

    void HandleSendComplete(Connection* connection, IOContext* io_context,
                           DWORD bytes_transferred) {
        delete io_context;

        // 큐에 다음 메시지가 있으면 전송
        std::lock_guard<std::mutex> lock(connection->send_mutex);
        if (!connection->send_queue.empty()) {
            auto& data = connection->send_queue.front();
            PostSend(connection, data.data(), data.size());
            connection->send_queue.pop();
        }
    }

    void HandleDisconnect(Connection* connection, IOContext* io_context) {
        closesocket(connection->socket);
        connections_.erase(connection->socket);
        delete io_context;
    }

private:
    HANDLE iocp_handle_;
    SOCKET listen_socket_;
    std::vector<std::thread> worker_threads_;
    std::unordered_map<SOCKET, std::unique_ptr<Connection>> connections_;
    std::atomic<bool> running_{true};
};

#endif // _WIN32
```

### BSD/macOS: kqueue

**특징:**
- epoll과 유사
- 파일, 소켓, 타이머, 시그널 이벤트 지원
- 효율적인 이벤트 알림

**구현:**

```cpp
// kqueue 기반 서버 (BSD/macOS)
#ifdef __APPLE__

class KqueueServer {
public:
    KqueueServer() {
        kqueue_fd_ = kqueue();
        if (kqueue_fd_ < 0) {
            throw std::runtime_error("kqueue failed");
        }
    }

    ~KqueueServer() {
        if (kqueue_fd_ >= 0) {
            close(kqueue_fd_);
        }
    }

    void Run(uint16_t port) {
        int listen_fd = CreateListenSocket(port);
        SetNonBlocking(listen_fd);

        // 리슨 소켓을 kqueue에 추가
        struct kevent event;
        EV_SET(&event, listen_fd, EVFILT_READ, EV_ADD, 0, 0, nullptr);
        kevent(kqueue_fd_, &event, 1, nullptr, 0, nullptr);

        constexpr int MAX_EVENTS = 1024;
        struct kevent events[MAX_EVENTS];

        while (running_) {
            // 이벤트 대기
            int num_events = kevent(kqueue_fd_, nullptr, 0, events, MAX_EVENTS, nullptr);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].ident;

                if (fd == listen_fd) {
                    // 새 연결
                    AcceptConnections(listen_fd);
                } else if (events[i].filter == EVFILT_READ) {
                    // 읽을 데이터 준비됨
                    HandleRead(fd);
                } else if (events[i].filter == EVFILT_WRITE) {
                    // 쓰기 준비됨
                    HandleWrite(fd);
                }

                if (events[i].flags & EV_EOF) {
                    // 연결 종료
                    HandleDisconnect(fd);
                }
            }
        }
    }

private:
    void AddSocket(int fd, int16_t filter) {
        struct kevent event;
        EV_SET(&event, fd, filter, EV_ADD, 0, 0, nullptr);
        kevent(kqueue_fd_, &event, 1, nullptr, 0, nullptr);
    }

    // epoll 구현과 유사...
    void AcceptConnections(int listen_fd);
    void HandleRead(int fd);
    void HandleWrite(int fd);
    void HandleDisconnect(int fd);

private:
    int kqueue_fd_;
    std::atomic<bool> running_{true};
    std::unordered_map<int, std::unique_ptr<Connection>> connections_;
};

#endif // __APPLE__
```

## Threading 전략

### 전략 1: 단일 I/O Thread + Game Thread

```
┌──────────────┐         ┌──────────────────┐
│  I/O Thread  │────────▶│   Game Thread    │
│              │  Queue  │                  │
│ • epoll/IOCP │         │ • 패킷 처리      │
│ • Recv/Send  │◀────────│ • 게임 업데이트   │
│              │  Queue  │ • 출력 생성       │
└──────────────┘         └──────────────────┘

장점: 단순함, 명확한 분리
단점: I/O가 병목이 될 수 있음, 단일 game thread 한계
```

### 전략 2: 다중 I/O Thread + Game Thread

```
┌─────────┐  ┌─────────┐
│ I/O 1   │  │ I/O 2   │
│(1000    │  │(1000    │
│ clients)│  │ clients)│
└────┬────┘  └────┬────┘
     │            │
     └─────┬──────┘
           │
           ▼
     ┌──────────┐
     │  Queue   │
     └─────┬────┘
           │
           ▼
     ┌──────────────┐
     │ Game Thread  │
     └──────────────┘

장점: 여러 코어로 I/O 확장
단점: Game thread가 여전히 병목
```

**구현:**

```cpp
class MultiIOThreadServer {
public:
    MultiIOThreadServer(int num_io_threads = 4)
        : num_io_threads_(num_io_threads) {}

    void Start() {
        running_ = true;

        // I/O thread 시작
        for (int i = 0; i < num_io_threads_; ++i) {
            io_threads_.emplace_back([this, i]() {
                IOThread(i);
            });
        }

        // Game thread 시작
        game_thread_ = std::thread([this]() {
            GameThread();
        });
    }

    void Stop() {
        running_ = false;
        for (auto& thread : io_threads_) {
            if (thread.joinable()) thread.join();
        }
        if (game_thread_.joinable()) game_thread_.join();
    }

private:
    void IOThread(int thread_id) {
        // 각 I/O thread는 자체 epoll 인스턴스를 가짐
        int epoll_fd = epoll_create1(0);

        // 연결을 I/O thread에 분배
        while (running_) {
            struct epoll_event events[256];
            int num_events = epoll_wait(epoll_fd, events, 256, 100);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].data.fd;

                if (events[i].events & EPOLLIN) {
                    // 데이터 읽기
                    char buffer[4096];
                    ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                    if (bytes > 0) {
                        // Game thread로 전달
                        Packet packet;
                        packet.connection_id = fd;
                        packet.data.assign(buffer, buffer + bytes);
                        input_queue_.push(packet);
                    }
                }

                if (events[i].events & EPOLLOUT) {
                    // 대기 중인 데이터 전송
                    SendPendingData(fd);
                }
            }

            // 나가는 패킷 전송
            Packet output;
            while (output_queue_.try_pop(output)) {
                send(output.connection_id, output.data.data(), output.data.size(), 0);
            }
        }

        close(epoll_fd);
    }

    void GameThread() {
        auto tick_interval = std::chrono::milliseconds(16);
        auto next_tick = std::chrono::steady_clock::now();

        while (running_) {
            // 모든 입력 패킷 처리
            Packet packet;
            while (input_queue_.try_pop(packet)) {
                ProcessPacket(packet);
            }

            // 게임 업데이트
            UpdateGame(0.016f);

            // 출력 생성
            auto outputs = GenerateOutputs();
            for (auto& output : outputs) {
                output_queue_.push(output);
            }

            next_tick += tick_interval;
            std::this_thread::sleep_until(next_tick);
        }
    }

private:
    int num_io_threads_;
    std::vector<std::thread> io_threads_;
    std::thread game_thread_;

    LockFreeQueue<Packet> input_queue_;
    LockFreeQueue<Packet> output_queue_;

    std::atomic<bool> running_;
};
```

### 전략 3: I/O Thread Pool + Game Thread Pool

```
┌────────────────────────────────┐
│    I/O Thread Pool (4)         │
│  • 연결 분배                    │
│  • 비동기 I/O (epoll/IOCP)     │
└────────┬───────────────────────┘
         │
         ▼
    ┌────────┐
    │ Queue  │
    └────┬───┘
         │
         ▼
┌────────────────────────────────┐
│   Game Thread Pool (4)         │
│  • 병렬 zone 업데이트           │
│  • Job system                  │
└────────────────────────────────┘

장점: 전체 CPU 활용
단점: 복잡한 동기화
```

## 프로토콜 설계

### 바이너리 프로토콜

```cpp
// 효율적인 바이너리 프로토콜
struct PacketHeader {
    uint16_t packet_id;
    uint16_t length;
    uint32_t sequence;
} __attribute__((packed));

struct PlayerMovePacket {
    PacketHeader header;
    uint32_t player_id;
    float x, y, z;
    float vx, vy, vz;
} __attribute__((packed));

// 직렬화
std::vector<char> SerializeMove(uint32_t player_id, const Vector3& pos, const Vector3& vel) {
    PlayerMovePacket packet;
    packet.header.packet_id = PACKET_PLAYER_MOVE;
    packet.header.length = sizeof(PlayerMovePacket);
    packet.header.sequence = next_sequence_++;
    packet.player_id = player_id;
    packet.x = pos.x; packet.y = pos.y; packet.z = pos.z;
    packet.vx = vel.x; packet.vy = vel.y; packet.vz = vel.z;

    std::vector<char> buffer(sizeof(packet));
    memcpy(buffer.data(), &packet, sizeof(packet));
    return buffer;
}

// 역직렬화
void ProcessPacket(const char* data, size_t length) {
    if (length < sizeof(PacketHeader)) return;

    const auto* header = reinterpret_cast<const PacketHeader*>(data);

    switch (header->packet_id) {
        case PACKET_PLAYER_MOVE: {
            if (length < sizeof(PlayerMovePacket)) return;

            const auto* move_packet = reinterpret_cast<const PlayerMovePacket*>(data);
            HandlePlayerMove(move_packet->player_id,
                Vector3{move_packet->x, move_packet->y, move_packet->z},
                Vector3{move_packet->vx, move_packet->vy, move_packet->vz});
            break;
        }
        // 다른 패킷 유형들...
    }
}
```

### 메시지 프레이밍

```cpp
// TCP 메시지 프레이밍 (길이 접두사 방식)
class MessageFramer {
public:
    void OnDataReceived(const char* data, size_t length) {
        buffer_.append(data, length);

        while (true) {
            // 길이를 위해 최소 4바이트 필요
            if (buffer_.size() < 4) break;

            // 메시지 길이 읽기
            uint32_t msg_length;
            memcpy(&msg_length, buffer_.data(), 4);
            msg_length = ntohl(msg_length); // 네트워크 바이트 순서

            // 완전한 메시지가 사용 가능한지 확인
            if (buffer_.size() < 4 + msg_length) break;

            // 메시지 추출
            std::string message(buffer_.data() + 4, msg_length);

            // 메시지 처리
            OnMessageComplete(message);

            // 버퍼에서 제거
            buffer_.erase(0, 4 + msg_length);
        }
    }

    std::vector<char> FrameMessage(const std::string& message) {
        uint32_t length = htonl(message.size());

        std::vector<char> framed;
        framed.resize(4 + message.size());

        memcpy(framed.data(), &length, 4);
        memcpy(framed.data() + 4, message.data(), message.size());

        return framed;
    }

private:
    std::string buffer_;
};
```

## 성능 최적화

### 1. Zero-Copy 기법

```cpp
// sendfile()을 사용한 zero-copy 파일 전송 (Linux)
void SendFileZeroCopy(int socket_fd, int file_fd, size_t length) {
    off_t offset = 0;
    while (offset < length) {
        ssize_t sent = sendfile(socket_fd, file_fd, &offset, length - offset);
        if (sent < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 소켓이 준비될 때까지 대기
                continue;
            }
            break;
        }
    }
}

// splice()를 사용한 zero-copy 파이프-소켓 전송 (Linux)
void SpliceData(int in_fd, int out_fd, size_t length) {
    while (length > 0) {
        ssize_t spliced = splice(in_fd, nullptr, out_fd, nullptr, length, SPLICE_F_MOVE);
        if (spliced <= 0) break;
        length -= spliced;
    }
}
```

### 2. 버퍼 풀링

```cpp
// 할당을 줄이기 위한 버퍼 풀
class BufferPool {
public:
    BufferPool(size_t buffer_size, size_t pool_size)
        : buffer_size_(buffer_size) {
        for (size_t i = 0; i < pool_size; ++i) {
            free_buffers_.push(std::make_unique<std::vector<char>>(buffer_size));
        }
    }

    std::unique_ptr<std::vector<char>> Acquire() {
        std::unique_lock<std::mutex> lock(mutex_);
        if (free_buffers_.empty()) {
            return std::make_unique<std::vector<char>>(buffer_size_);
        }

        auto buffer = std::move(free_buffers_.front());
        free_buffers_.pop();
        return buffer;
    }

    void Release(std::unique_ptr<std::vector<char>> buffer) {
        buffer->clear();
        std::lock_guard<std::mutex> lock(mutex_);
        free_buffers_.push(std::move(buffer));
    }

private:
    size_t buffer_size_;
    std::queue<std::unique_ptr<std::vector<char>>> free_buffers_;
    std::mutex mutex_;
};

// 사용 예시
BufferPool buffer_pool(4096, 1000); // 4KB 버퍼 1000개

void HandleConnection(int fd) {
    auto buffer = buffer_pool.Acquire();
    ssize_t bytes = recv(fd, buffer->data(), buffer->size(), 0);
    ProcessData(buffer->data(), bytes);
    buffer_pool.Release(std::move(buffer));
}
```

### 3. TCP 튜닝

```cpp
// 게임 서버를 위한 TCP 설정 최적화
void OptimizeTCPSocket(int socket_fd) {
    // Nagle 알고리즘 비활성화 (지연 시간 감소)
    int flag = 1;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

    // 송신/수신 버퍼 크기 설정
    int buffer_size = 256 * 1024; // 256 KB
    setsockopt(socket_fd, SOL_SOCKET, SO_SNDBUF, &buffer_size, sizeof(buffer_size));
    setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &buffer_size, sizeof(buffer_size));

    // TCP keepalive 활성화
    flag = 1;
    setsockopt(socket_fd, SOL_SOCKET, SO_KEEPALIVE, &flag, sizeof(flag));

    // keepalive 매개변수 설정 (Linux)
    int keepalive_time = 60;      // 60초
    int keepalive_interval = 10;   // 10초
    int keepalive_probes = 3;      // 3회 프로브

    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepalive_time, sizeof(keepalive_time));
    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepalive_interval, sizeof(keepalive_interval));
    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPCNT, &keepalive_probes, sizeof(keepalive_probes));
}
```

## 실제 구현 사례

### Nginx 아키텍처

```
Nginx 프로세스 모델
┌──────────────────────────────────┐
│       Master Process             │
│  • 설정 관리                      │
│  • 워커 관리                      │
└────────┬─────────────────────────┘
         │
    ┌────┴─────┬──────┬──────┐
    │          │      │      │
    ▼          ▼      ▼      ▼
┌────────┐ ┌────────┐ ... ┌────────┐
│Worker 1│ │Worker 2│     │Worker N│
│        │ │        │     │        │
│ • epoll│ │ • epoll│     │ • epoll│
│ • 1K   │ │ • 1K   │     │ • 1K   │
│   conn │ │   conn │     │   conn │
└────────┘ └────────┘     └────────┘

각 워커: 단일 thread event loop
연결 분배: SO_REUSEPORT 사용
```

### Redis 아키텍처

```
Redis Event Loop
┌──────────────────────────────────┐
│    단일 thread Event Loop        │
├──────────────────────────────────┤
│  ┌────────────────────────────┐  │
│  │   File Event Handler      │  │
│  │  (클라이언트 연결)          │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │   Time Event Handler      │  │
│  │  (주기적 작업)              │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │    명령 처리                │  │
│  │  (인메모리 연산)            │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘

장점: 단순함, 락 없음, 빠름
단점: 단일 코어만 사용
```

### Node.js (libuv)

```
libuv 아키텍처
┌────────────────────────────────────┐
│         Event Loop Thread          │
│  ┌──────────────────────────────┐  │
│  │  1. 타이머                    │  │
│  │  2. 대기 중인 콜백            │  │
│  │  3. Idle, prepare             │  │
│  │  4. Poll (I/O)                │  │
│  │  5. Check                     │  │
│  │  6. Close 콜백                │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│       Thread Pool (4 threads)      │
│  • 파일 I/O                        │
│  • DNS 조회                        │
│  • CPU 집약적 작업                  │
└────────────────────────────────────┘

JavaScript: 단일 thread
I/O 및 블로킹 작업: Thread pool
```

## 결론

효율적인 네트워크 I/O는 게임 서버 성능에 매우 중요합니다. 현대 서버는 비동기 I/O (epoll, IOCP, kqueue)를 사용하여 최소한의 thread로 수천 개의 동시 연결을 처리합니다. Threading 전략의 선택은 게임 요구사항에 따라 달라집니다: 단일 thread event loop은 간단한 서버에 적합하고, 복잡한 MMO는 다중 I/O thread와 game thread pool의 이점을 누릴 수 있습니다. 프로토콜 설계, TCP 튜닝, 그리고 zero-copy 기법은 성능을 더욱 최적화합니다.

## 추가 읽을거리

- [The C10K Problem](http://www.kegel.com/c10k.html)
- [epoll vs IOCP](https://github.com/spotify/netty-zmtp/blob/master/doc/epoll-iocp.md)
- [Boost.Asio Documentation](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
- [libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html)
- [Nginx Architecture](https://www.aosabook.org/en/nginx.html)
