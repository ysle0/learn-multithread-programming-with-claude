# Networking I/O and Threading

## Overview

Network I/O is often the bottleneck in game servers. Efficiently handling thousands of concurrent connections requires advanced I/O models and threading strategies. This document covers asynchronous I/O, I/O completion ports (IOCP), epoll, thread pools for network handling, and zero-copy techniques.

## Table of Contents

1. [I/O Models](#io-models)
2. [Platform-Specific APIs](#platform-specific-apis)
3. [Threading Strategies](#threading-strategies)
4. [Protocol Design](#protocol-design)
5. [Performance Optimization](#performance-optimization)
6. [Real-world Implementations](#real-world-implementations)

## I/O Models

### 1. Blocking I/O

**Concept:** Thread blocks until I/O operation completes.

```
Thread Timeline (Blocking I/O)
─────────────────────────────────────
Thread 1: [recv]─────────[recv]─────
           ↓ blocks      ↓ blocks
          waiting       waiting

Pros: Simple
Cons: One thread per connection, doesn't scale
```

**Implementation:**

```cpp
// Blocking I/O - Simple but doesn't scale
void HandleClient(int socket_fd) {
    char buffer[4096];

    while (true) {
        // Blocks until data arrives
        ssize_t bytes_read = recv(socket_fd, buffer, sizeof(buffer), 0);

        if (bytes_read <= 0) {
            break; // Connection closed or error
        }

        // Process data
        ProcessData(buffer, bytes_read);

        // Send response (also blocks)
        send(socket_fd, response, response_length, 0);
    }

    close(socket_fd);
}

// Thread per connection (doesn't scale beyond ~1000 connections)
void RunServer() {
    int listen_fd = CreateListenSocket(8080);

    while (true) {
        int client_fd = accept(listen_fd, nullptr, nullptr);

        // Spawn thread for each connection
        std::thread client_thread(HandleClient, client_fd);
        client_thread.detach();
    }
}
```

### 2. Non-blocking I/O with select/poll

**Concept:** Single thread monitors multiple sockets.

```
Event Loop (select/poll)
──────────────────────────────────────
         ┌─────────────┐
         │  select()   │───────┐
         └─────────────┘       │
                │              │
         Ready sockets         │
                │              │
         ┌──────▼──────┐       │
         │ Process I/O │       │
         └─────────────┘       │
                │              │
                └──────────────┘

Pros: Handles multiple connections
Cons: O(n) to scan FDs, doesn't scale to 10K+ connections
```

**Implementation:**

```cpp
// Non-blocking I/O with select
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

            // Wait for activity on any socket
            int activity = select(max_fd + 1, &read_set, nullptr, nullptr, nullptr);

            if (activity < 0) {
                perror("select");
                break;
            }

            // Check each socket
            for (int fd = 0; fd <= max_fd; ++fd) {
                if (!FD_ISSET(fd, &read_set)) {
                    continue;
                }

                if (fd == listen_fd) {
                    // New connection
                    int client_fd = accept(listen_fd, nullptr, nullptr);
                    SetNonBlocking(client_fd);
                    FD_SET(client_fd, &master_set);
                    max_fd = std::max(max_fd, client_fd);
                } else {
                    // Data from existing connection
                    char buffer[4096];
                    ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                    if (bytes <= 0) {
                        // Connection closed
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

### 3. Asynchronous I/O (Reactor Pattern)

**Concept:** Event-driven I/O with callbacks.

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

Pros: Scalable, efficient
Cons: Callback complexity
```

### 4. Proactor Pattern (Asynchronous Completion)

**Concept:** I/O operations complete asynchronously, callbacks invoked on completion.

```
Proactor Pattern (IOCP)
─────────────────────────────────────
  Initiate I/O
       │
       ▼
  ┌─────────────┐
  │   Kernel    │
  │  (Async I/O)│
  └──────┬──────┘
         │
    Completion
         │
         ▼
  ┌─────────────┐
  │ Completion  │
  │   Handler   │
  └─────────────┘

Pros: True async, scalable
Cons: Complex, platform-specific
```

## Platform-Specific APIs

### Linux: epoll

**Characteristics:**
- Edge-triggered and level-triggered modes
- O(1) performance for ready events
- Scales to 100K+ connections

**Implementation:**

```cpp
// epoll-based server (Linux)
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

        // Add listen socket to epoll
        AddSocket(listen_fd, EPOLLIN | EPOLLET); // Edge-triggered

        constexpr int MAX_EVENTS = 1024;
        struct epoll_event events[MAX_EVENTS];

        while (running_) {
            // Wait for events
            int num_events = epoll_wait(epoll_fd_, events, MAX_EVENTS, -1);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].data.fd;

                if (fd == listen_fd) {
                    // Accept new connections
                    AcceptConnections(listen_fd);
                } else if (events[i].events & EPOLLIN) {
                    // Data ready to read
                    HandleRead(fd);
                } else if (events[i].events & EPOLLOUT) {
                    // Ready to write
                    HandleWrite(fd);
                } else if (events[i].events & (EPOLLHUP | EPOLLERR)) {
                    // Connection error or closed
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
                    // No more connections
                    break;
                } else {
                    perror("accept");
                    break;
                }
            }

            SetNonBlocking(client_fd);
            AddSocket(client_fd, EPOLLIN | EPOLLOUT | EPOLLET);

            // Create connection state
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
                    // No more data
                    break;
                } else {
                    // Error
                    HandleDisconnect(fd);
                    return;
                }
            } else if (bytes == 0) {
                // Connection closed
                HandleDisconnect(fd);
                return;
            }

            // Process data
            conn->input_buffer.append(buffer, bytes);
        }

        // Process complete packets
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
                    // Would block, try later
                    break;
                } else {
                    // Error
                    HandleDisconnect(fd);
                    return;
                }
            }

            // Remove sent data from buffer
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

**Characteristics:**
- Completion-based (Proactor pattern)
- Scales to 100K+ connections
- Integrates with thread pool

**Implementation:**

```cpp
// IOCP-based server (Windows)
#ifdef _WIN32

class IOCPServer {
public:
    IOCPServer(int num_threads = 0) {
        // Create I/O completion port
        iocp_handle_ = CreateIoCompletionPort(
            INVALID_HANDLE_VALUE,
            nullptr,
            0,
            num_threads // 0 = number of processors
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
        // Create listen socket
        listen_socket_ = CreateListenSocket(port);

        // Associate listen socket with IOCP
        CreateIoCompletionPort(
            (HANDLE)listen_socket_,
            iocp_handle_,
            (ULONG_PTR)nullptr,
            0
        );

        // Start worker threads
        int num_threads = std::thread::hardware_concurrency();
        for (int i = 0; i < num_threads; ++i) {
            worker_threads_.emplace_back([this]() {
                WorkerThread();
            });
        }

        // Accept connections
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

            // Wait for I/O completion
            BOOL result = GetQueuedCompletionStatus(
                iocp_handle_,
                &bytes_transferred,
                &completion_key,
                &overlapped,
                INFINITE
            );

            if (!overlapped) {
                // Shutdown signal
                break;
            }

            auto* io_context = CONTAINING_RECORD(
                overlapped,
                IOContext,
                overlapped
            );

            auto* connection = reinterpret_cast<Connection*>(completion_key);

            if (!result || bytes_transferred == 0) {
                // Connection closed or error
                HandleDisconnect(connection, io_context);
                continue;
            }

            // Handle I/O completion
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
            // Post accept operation
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
                0, // No initial receive
                sizeof(sockaddr_in) + 16,
                sizeof(sockaddr_in) + 16,
                &bytes_received,
                &io_context->overlapped
            );

            if (!result && WSAGetLastError() != ERROR_IO_PENDING) {
                // Error
                closesocket(io_context->socket);
                delete io_context;
            }

            // Wait a bit before next accept
            std::this_thread::sleep_for(std::chrono::milliseconds(1));
        }
    }

    void HandleAcceptComplete(IOContext* io_context, DWORD bytes_transferred) {
        // Create connection
        auto connection = std::make_unique<Connection>();
        connection->socket = io_context->socket;

        // Associate socket with IOCP
        CreateIoCompletionPort(
            (HANDLE)connection->socket,
            iocp_handle_,
            (ULONG_PTR)connection.get(),
            0
        );

        connections_[connection->socket] = std::move(connection);

        // Post initial receive
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
            // Error
            delete io_context;
        }
    }

    void HandleReceiveComplete(Connection* connection, IOContext* io_context,
                               DWORD bytes_transferred) {
        // Process received data
        ProcessData(connection, io_context->buffer, bytes_transferred);

        // Post another receive
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
            // Error
            delete io_context;
        }
    }

    void HandleSendComplete(Connection* connection, IOContext* io_context,
                           DWORD bytes_transferred) {
        delete io_context;

        // Send next queued message if any
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

**Characteristics:**
- Similar to epoll
- Supports file, socket, timer, signal events
- Efficient event notification

**Implementation:**

```cpp
// kqueue-based server (BSD/macOS)
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

        // Add listen socket to kqueue
        struct kevent event;
        EV_SET(&event, listen_fd, EVFILT_READ, EV_ADD, 0, 0, nullptr);
        kevent(kqueue_fd_, &event, 1, nullptr, 0, nullptr);

        constexpr int MAX_EVENTS = 1024;
        struct kevent events[MAX_EVENTS];

        while (running_) {
            // Wait for events
            int num_events = kevent(kqueue_fd_, nullptr, 0, events, MAX_EVENTS, nullptr);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].ident;

                if (fd == listen_fd) {
                    // New connection
                    AcceptConnections(listen_fd);
                } else if (events[i].filter == EVFILT_READ) {
                    // Data ready to read
                    HandleRead(fd);
                } else if (events[i].filter == EVFILT_WRITE) {
                    // Ready to write
                    HandleWrite(fd);
                }

                if (events[i].flags & EV_EOF) {
                    // Connection closed
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

    // Similar to epoll implementation...
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

## Threading Strategies

### Strategy 1: Single I/O Thread + Game Thread

```
┌──────────────┐         ┌──────────────────┐
│  I/O Thread  │────────▶│   Game Thread    │
│              │  Queue  │                  │
│ • epoll/IOCP │         │ • Process packets│
│ • Recv/Send  │◀────────│ • Update game    │
│              │  Queue  │ • Generate output│
└──────────────┘         └──────────────────┘

Pros: Simple, clear separation
Cons: I/O can bottleneck, single game thread limit
```

### Strategy 2: Multiple I/O Threads + Game Thread

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

Pros: Scale I/O to multiple cores
Cons: Game thread still bottleneck
```

**Implementation:**

```cpp
class MultiIOThreadServer {
public:
    MultiIOThreadServer(int num_io_threads = 4)
        : num_io_threads_(num_io_threads) {}

    void Start() {
        running_ = true;

        // Start I/O threads
        for (int i = 0; i < num_io_threads_; ++i) {
            io_threads_.emplace_back([this, i]() {
                IOThread(i);
            });
        }

        // Start game thread
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
        // Each I/O thread has its own epoll instance
        int epoll_fd = epoll_create1(0);

        // Distribute connections across I/O threads
        while (running_) {
            struct epoll_event events[256];
            int num_events = epoll_wait(epoll_fd, events, 256, 100);

            for (int i = 0; i < num_events; ++i) {
                int fd = events[i].data.fd;

                if (events[i].events & EPOLLIN) {
                    // Read data
                    char buffer[4096];
                    ssize_t bytes = recv(fd, buffer, sizeof(buffer), 0);

                    if (bytes > 0) {
                        // Push to game thread
                        Packet packet;
                        packet.connection_id = fd;
                        packet.data.assign(buffer, buffer + bytes);
                        input_queue_.push(packet);
                    }
                }

                if (events[i].events & EPOLLOUT) {
                    // Write pending data
                    SendPendingData(fd);
                }
            }

            // Send outgoing packets
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
            // Process all input packets
            Packet packet;
            while (input_queue_.try_pop(packet)) {
                ProcessPacket(packet);
            }

            // Update game
            UpdateGame(0.016f);

            // Generate outputs
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

### Strategy 3: I/O Thread Pool + Game Thread Pool

```
┌────────────────────────────────┐
│    I/O Thread Pool (4)         │
│  • Distribute connections      │
│  • Async I/O (epoll/IOCP)      │
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
│  • Parallel zone updates       │
│  • Job system                  │
└────────────────────────────────┘

Pros: Full CPU utilization
Cons: Complex synchronization
```

## Protocol Design

### Binary Protocol

```cpp
// Efficient binary protocol
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

// Serialization
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

// Deserialization
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
        // Other packet types...
    }
}
```

### Message Framing

```cpp
// TCP message framing (length-prefixed)
class MessageFramer {
public:
    void OnDataReceived(const char* data, size_t length) {
        buffer_.append(data, length);

        while (true) {
            // Need at least 4 bytes for length
            if (buffer_.size() < 4) break;

            // Read message length
            uint32_t msg_length;
            memcpy(&msg_length, buffer_.data(), 4);
            msg_length = ntohl(msg_length); // Network byte order

            // Check if complete message is available
            if (buffer_.size() < 4 + msg_length) break;

            // Extract message
            std::string message(buffer_.data() + 4, msg_length);

            // Process message
            OnMessageComplete(message);

            // Remove from buffer
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

## Performance Optimization

### 1. Zero-Copy Techniques

```cpp
// sendfile() for zero-copy file transmission (Linux)
void SendFileZeroCopy(int socket_fd, int file_fd, size_t length) {
    off_t offset = 0;
    while (offset < length) {
        ssize_t sent = sendfile(socket_fd, file_fd, &offset, length - offset);
        if (sent < 0) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // Wait for socket to be ready
                continue;
            }
            break;
        }
    }
}

// splice() for zero-copy pipe-to-socket (Linux)
void SpliceData(int in_fd, int out_fd, size_t length) {
    while (length > 0) {
        ssize_t spliced = splice(in_fd, nullptr, out_fd, nullptr, length, SPLICE_F_MOVE);
        if (spliced <= 0) break;
        length -= spliced;
    }
}
```

### 2. Buffer Pooling

```cpp
// Buffer pool to reduce allocations
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

// Usage
BufferPool buffer_pool(4096, 1000); // 1000 buffers of 4KB each

void HandleConnection(int fd) {
    auto buffer = buffer_pool.Acquire();
    ssize_t bytes = recv(fd, buffer->data(), buffer->size(), 0);
    ProcessData(buffer->data(), bytes);
    buffer_pool.Release(std::move(buffer));
}
```

### 3. TCP Tuning

```cpp
// Optimize TCP settings for game servers
void OptimizeTCPSocket(int socket_fd) {
    // Disable Nagle's algorithm (reduces latency)
    int flag = 1;
    setsockopt(socket_fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

    // Set send/receive buffer sizes
    int buffer_size = 256 * 1024; // 256 KB
    setsockopt(socket_fd, SOL_SOCKET, SO_SNDBUF, &buffer_size, sizeof(buffer_size));
    setsockopt(socket_fd, SOL_SOCKET, SO_RCVBUF, &buffer_size, sizeof(buffer_size));

    // Enable TCP keepalive
    flag = 1;
    setsockopt(socket_fd, SOL_SOCKET, SO_KEEPALIVE, &flag, sizeof(flag));

    // Set keepalive parameters (Linux)
    int keepalive_time = 60;      // 60 seconds
    int keepalive_interval = 10;   // 10 seconds
    int keepalive_probes = 3;      // 3 probes

    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepalive_time, sizeof(keepalive_time));
    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepalive_interval, sizeof(keepalive_interval));
    setsockopt(socket_fd, IPPROTO_TCP, TCP_KEEPCNT, &keepalive_probes, sizeof(keepalive_probes));
}
```

## Real-world Implementations

### Nginx Architecture

```
Nginx Process Model
┌──────────────────────────────────┐
│       Master Process             │
│  • Configuration                 │
│  • Worker management             │
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

Each worker: Single-threaded event loop
Connections: Distributed via SO_REUSEPORT
```

### Redis Architecture

```
Redis Event Loop
┌──────────────────────────────────┐
│    Single-threaded Event Loop    │
├──────────────────────────────────┤
│  ┌────────────────────────────┐  │
│  │     File Event Handler     │  │
│  │  (Client connections)      │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │     Time Event Handler     │  │
│  │  (Periodic tasks)          │  │
│  └────────────────────────────┘  │
│  ┌────────────────────────────┐  │
│  │    Command Processing      │  │
│  │  (In-memory operations)    │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘

Pros: Simple, no locks, fast
Cons: Single core only
```

### Node.js (libuv)

```
libuv Architecture
┌────────────────────────────────────┐
│         Event Loop Thread          │
│  ┌──────────────────────────────┐  │
│  │  1. Timers                   │  │
│  │  2. Pending callbacks        │  │
│  │  3. Idle, prepare            │  │
│  │  4. Poll (I/O)               │  │
│  │  5. Check                    │  │
│  │  6. Close callbacks          │  │
│  └──────────────────────────────┘  │
└────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────┐
│       Thread Pool (4 threads)      │
│  • File I/O                        │
│  • DNS lookups                     │
│  • CPU-intensive tasks             │
└────────────────────────────────────┘

JavaScript: Single-threaded
I/O & blocking tasks: Thread pool
```

## Conclusion

Efficient network I/O is critical for game server performance. Modern servers use asynchronous I/O (epoll, IOCP, kqueue) to handle thousands of concurrent connections with minimal threads. The choice of threading strategy depends on game requirements: single-threaded event loops work for simple servers, while complex MMOs benefit from multiple I/O threads and game thread pools. Protocol design, TCP tuning, and zero-copy techniques further optimize performance.

## Further Reading

- [The C10K Problem](http://www.kegel.com/c10k.html)
- [epoll vs IOCP](https://github.com/spotify/netty-zmtp/blob/master/doc/epoll-iocp.md)
- [Boost.Asio Documentation](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html)
- [libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html)
- [Nginx Architecture](https://www.aosabook.org/en/nginx.html)
