# 게임 서버 애플리케이션

## 개요

게임 서버는 멀티플레이어 게임 세션을 관리하고, 플레이어 연결을 처리하며, 게임 상태를 동기화하고, 공정한 게임플레이를 보장하는 특수한 네트워크 애플리케이션입니다. 수천 명의 동시 접속 플레이어를 처리할 수 있는 고성능, 확장 가능한 게임 서버를 구축하려면 멀티스레딩과 동시성에 대한 이해가 필수적입니다.

## 목차

1. [실시간 게임 서버](01-realtime-game-server.md) - MMO, MOBA, FPS 아키텍처
2. [비실시간 서버](02-non-realtime-server.md) - 웹 서버, REST API
3. [MMO 아키텍처](03-mmo-architecture.md) - sharding, 월드 파티셔닝
4. [Game Loop 스레딩](04-game-loop-threading.md) - game loop와 멀티스레딩
5. [네트워킹 스레딩](05-networking-threading.md) - 네트워킹 I/O와 스레딩

## 게임 서버 아키텍처 패턴

### 상위 수준 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     GAME SERVER CLUSTER                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Gateway    │  │   Gateway    │  │   Gateway    │      │
│  │   Server     │  │   Server     │  │   Server     │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │               │
│         └──────────────────┼──────────────────┘               │
│                            │                                  │
│  ┌─────────────────────────┴─────────────────────────┐       │
│  │              Load Balancer / Router                │       │
│  └─────────────────────────┬─────────────────────────┘       │
│                            │                                  │
│    ┌───────────────────────┼───────────────────────┐         │
│    │                       │                       │         │
│  ┌─┴────────┐    ┌─────────┴──────┐    ┌──────────┴───┐    │
│  │  World   │    │     World      │    │    World     │    │
│  │ Server 1 │    │    Server 2    │    │   Server 3   │    │
│  │ (Zone A) │    │    (Zone B)    │    │   (Zone C)   │    │
│  └─────┬────┘    └────────┬───────┘    └──────┬───────┘    │
│        │                  │                    │             │
│        └──────────────────┼────────────────────┘             │
│                           │                                  │
│  ┌────────────────────────┴────────────────────────┐        │
│  │              Shared Services Layer              │        │
│  ├─────────────────────────────────────────────────┤        │
│  │ • Database Cluster (Player Data, Game State)    │        │
│  │ • Cache Layer (Redis, Memcached)                │        │
│  │ • Message Queue (RabbitMQ, Kafka)               │        │
│  │ • Matchmaking Service                           │        │
│  │ • Authentication Service                        │        │
│  │ • Analytics & Monitoring                        │        │
│  └─────────────────────────────────────────────────┘        │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

## 스레딩 모델 비교

### 1. 단일 스레드 event loop
```
┌──────────────────────────────────┐
│     Main Thread (Event Loop)     │
├──────────────────────────────────┤
│ • Network I/O (non-blocking)     │
│ • Game Logic                     │
│ • State Updates                  │
│ • Timer Events                   │
└──────────────────────────────────┘

장점: 단순함, 동기화 불필요
단점: 제한된 CPU 활용
사용 사례: 소규모 게임, 턴 기반
```

### 2. Thread-per-Connection
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Thread 1   │  │  Thread 2   │  │  Thread N   │
│ (Player 1)  │  │ (Player 2)  │  │ (Player N)  │
├─────────────┤  ├─────────────┤  ├─────────────┤
│ • Network   │  │ • Network   │  │ • Network   │
│ • Logic     │  │ • Logic     │  │ • Logic     │
│ • State     │  │ • State     │  │ • State     │
└─────────────┘  └─────────────┘  └─────────────┘

장점: 단순한 동시성 모델
단점: 스레드 오버헤드, 확장성 부족
사용 사례: 레거시 서버, 1000명 미만 플레이어
```

### 3. Thread Pool 아키텍처
```
┌───────────────────────────────────────────────────┐
│            Network I/O Thread Pool                │
│  (Async I/O - IOCP/epoll)                        │
└─────────────────┬─────────────────────────────────┘
                  │
                  ▼
┌───────────────────────────────────────────────────┐
│              Task Queue (Lock-free)               │
└─────────────────┬─────────────────────────────────┘
                  │
    ┌─────────────┼─────────────┬─────────────┐
    │             │             │             │
    ▼             ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│Worker 1 │  │Worker 2 │  │Worker 3 │  │Worker N │
│ (Logic) │  │ (Logic) │  │ (Logic) │  │ (Logic) │
└─────────┘  └─────────┘  └─────────┘  └─────────┘

장점: 우수한 CPU 활용, 확장 가능
단점: 복잡한 동기화
사용 사례: 현대 게임 서버
```

### 4. Actor 모델
```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│   Actor 1   │  │   Actor 2   │  │   Actor N   │
│  (Player)   │  │   (Zone)    │  │  (Entity)   │
├─────────────┤  ├─────────────┤  ├─────────────┤
│ • Mailbox   │  │ • Mailbox   │  │ • Mailbox   │
│ • State     │  │ • State     │  │ • State     │
│ • Behavior  │  │ • Behavior  │  │ • Behavior  │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
            ┌───────────┴───────────┐
            │   Message Dispatcher   │
            │    (Thread Pool)       │
            └───────────────────────┘

장점: 격리성, 내결함성
단점: 메시지 전달 오버헤드
사용 사례: Erlang/Elixir 서버, 분산 시스템
```

### 5. 데이터 지향 설계 (ECS)
```
┌────────────────────────────────────────────────┐
│              Component Arrays                  │
├────────────────────────────────────────────────┤
│ Position[] │ Velocity[] │ Health[] │ AI[]     │
└────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  Physics     │ │   Combat     │ │    AI        │
│  System      │ │   System     │ │   System     │
│  (Thread 1)  │ │  (Thread 2)  │ │  (Thread 3)  │
└──────────────┘ └──────────────┘ └──────────────┘

장점: 캐시 친화적, 병렬 처리
단점: 복잡한 아키텍처
사용 사례: 고성능 시뮬레이션
```

## 성능 특성

| 아키텍처            | 확장성      | 지연 시간   | 처리량     | 복잡도     |
|---------------------|-------------|---------|------------|------------|
| 단일 스레드          | ★☆☆☆☆       | ★★★★★   | ★☆☆☆☆      | ★★★★★      |
| Thread-per-conn     | ★★☆☆☆       | ★★★☆☆   | ★★☆☆☆      | ★★★☆☆      |
| Thread Pool         | ★★★★☆       | ★★★★☆   | ★★★★☆      | ★★★☆☆      |
| Actor 모델          | ★★★★★       | ★★★☆☆   | ★★★★☆      | ★★☆☆☆      |
| ECS                 | ★★★★★       | ★★★★★   | ★★★★★      | ★★☆☆☆      |

## 게임 서버 유형

### 실시간 게임 서버
- **FPS (1인칭 슈팅)**: 높은 tick rate (60-128 Hz), 낮은 지연 시간 필수
- **MOBA (멀티플레이어 온라인 배틀 아레나)**: 20-30 Hz tick rate, 결정론적 시뮬레이션
- **MMO (대규모 다중 접속 온라인)**: 가변 tick rate, zone 기반 업데이트
- **레이싱 게임**: 고빈도 물리 업데이트, 클라이언트 예측

### 비실시간 서버
- **턴 기반 전략**: 이벤트 기반, 엄격한 타이밍 요구 없음
- **카드 게임**: 요청-응답 모델, 로비 시스템
- **소셜 게임**: 웹 기반, REST API, stateless 서비스
- **비동기 멀티플레이어**: 지연된 턴, mailbox 시스템

## 핵심 성능 지표

### 지연 시간 목표

```
Game Type            | Target Latency | Acceptable Range
---------------------|----------------|------------------
FPS (Competitive)    | < 20ms         | 20-50ms
MOBA                 | < 30ms         | 30-80ms
MMO (Combat)         | < 50ms         | 50-150ms
MMO (Social)         | < 100ms        | 100-300ms
Turn-based           | < 200ms        | 200-1000ms
```

### Tick Rate 대 플레이어 수

```
            │
  Tick Rate │     FPS
  (Hz)      │      •
            │
    128     │      •
            │
     64     │        •   Racing
            │
     30     │            •  MOBA
            │
     20     │                 •
            │                   •  MMO (Combat)
     10     │                      •
            │                         •
      5     │                            •  MMO (World)
            │                               •
      1     │─────────────────────────────────•─────────
            0    100   1K   10K  50K  100K    1M
                      Concurrent Players
```

## 스레딩 모범 사례

### 1. I/O와 게임 로직 분리
```cpp
// 좋은 예: I/O 스레드와 게임 로직 분리
class GameServer {
    std::thread io_thread_;
    std::thread game_thread_;

    void IOThread() {
        while (running_) {
            // 네트워크 I/O 처리
            PollNetworkEvents();

            // 이벤트를 게임 스레드로 전달
            game_queue_.push(events);
        }
    }

    void GameThread() {
        while (running_) {
            // I/O에서 온 이벤트 처리
            ProcessEvents(game_queue_.pop());

            // 게임 상태 업데이트
            UpdateGameLogic(delta_time);

            // 네트워크 업데이트 준비
            PrepareNetworkUpdates();
        }
    }
};
```

### 2. Lock-free 자료 구조 사용
```cpp
// 스레드 간 통신을 위한 lock-free 큐
template<typename T>
class LockFreeQueue {
    std::atomic<Node*> head_;
    std::atomic<Node*> tail_;

public:
    void push(T value);
    bool try_pop(T& value);
};

// 사용 예시
LockFreeQueue<NetworkEvent> event_queue_;
```

### 3. 동기화 지점 최소화
```cpp
// 나쁜 예: 빈번한 잠금
void ProcessPlayers() {
    for (auto& player : players_) {
        std::lock_guard<std::mutex> lock(player.mutex);
        player.Update();
    }
}

// 좋은 예: 일괄 업데이트, 단일 잠금
void ProcessPlayers() {
    std::vector<PlayerUpdate> updates;

    // 잠금 없이 업데이트 수집
    for (auto& player : players_) {
        updates.push_back(player.PrepareUpdate());
    }

    // 단일 잠금으로 업데이트 적용
    std::lock_guard<std::mutex> lock(world_mutex_);
    for (auto& update : updates) {
        ApplyUpdate(update);
    }
}
```

### 4. Thread-local Storage 사용
```cpp
// 스레드 로컬 난수 생성기
thread_local std::mt19937 rng(std::random_device{}());

// 스레드 로컬 메모리 풀
thread_local MemoryPool<1024> pool;
```

## 확장 전략

### 수평 확장
```
┌─────────────────────────────────────────────────┐
│            Load Balancer (DNS/Nginx)            │
└────────┬──────────┬──────────┬─────────────────┘
         │          │          │
    ┌────┴───┐ ┌────┴───┐ ┌────┴───┐
    │Server 1│ │Server 2│ │Server N│
    │  1000  │ │  1000  │ │  1000  │
    │ players│ │ players│ │ players│
    └────┬───┘ └────┬───┘ └────┬───┘
         │          │          │
         └──────────┴──────────┴────────────┐
                                             │
                    ┌────────────────────────┴────┐
                    │  Shared Database / Cache    │
                    └─────────────────────────────┘
```

### 수직 확장 (멀티코어 최적화)
```cpp
// 모든 CPU 코어 활용
void OptimizeForMultiCore() {
    unsigned int num_threads = std::thread::hardware_concurrency();

    // I/O 스레드 (네트워크 인터페이스당 1-2개)
    size_t io_threads = 2;

    // 게임 로직 스레드 (나머지 코어)
    size_t logic_threads = num_threads - io_threads;

    ThreadPool io_pool(io_threads);
    ThreadPool logic_pool(logic_threads);
}
```

### Zone 기반 파티셔닝
```
World Map (MMO)
┌──────────────┬──────────────┬──────────────┐
│   Zone A     │   Zone B     │   Zone C     │
│  Server 1    │  Server 2    │  Server 3    │
│  Thread 1    │  Thread 2    │  Thread 3    │
│  1000 NPCs   │  1500 NPCs   │  800 NPCs    │
│  300 players │  450 players │  200 players │
└──────────────┴──────────────┴──────────────┘
│   Zone D     │   Zone E     │   Zone F     │
│  Server 4    │  Server 5    │  Server 6    │
└──────────────┴──────────────┴──────────────┘
```

## 일반적인 과제

### 1. 상태 동기화
- **문제**: 여러 스레드/서버 간 게임 상태 일관성 유지
- **해결책**:
  - 이벤트 소싱
  - CQRS (Command Query Responsibility Segregation)
  - 연산 변환
  - 결정론적 lockstep

### 2. 지연 보상
- **문제**: 네트워크 지연으로 인한 일관되지 않은 플레이어 경험
- **해결책**:
  - 클라이언트 측 예측
  - 서버 조정
  - 지연 보상
  - 관심 영역 관리

### 3. 부하 분산
- **문제**: 플레이어/zone의 불균등 분배
- **해결책**:
  - 동적 zone 분할/병합
  - 플레이어 마이그레이션
  - 인스턴싱
  - Consistent hashing

### 4. 데드락 방지
- **문제**: 여러 스레드가 서로를 기다리는 상태
- **해결책**:
  - 잠금 순서 지정
  - 타임아웃
  - Lock-free 알고리즘
  - Actor 모델 (공유 상태 없음)

## 기술 스택 예시

### C++ 스택 (고성능)
```
┌─────────────────────────────────────┐
│ Networking: Boost.Asio, libuv      │
│ Threading: std::thread, TBB        │
│ Lock-free: Folly, libcds           │
│ Serialization: Protobuf, FlatBuffers│
│ Physics: PhysX, Bullet             │
└─────────────────────────────────────┘
```

### Rust 스택 (안전한 동시성)
```
┌─────────────────────────────────────┐
│ Networking: Tokio, async-std       │
│ Concurrency: Rayon, crossbeam      │
│ Serialization: serde, bincode      │
│ Game engine: Bevy (ECS)            │
└─────────────────────────────────────┘
```

### Go 스택 (확장성)
```
┌─────────────────────────────────────┐
│ Networking: net/http, gRPC         │
│ Concurrency: Goroutines, channels  │
│ Game framework: Pitaya, Nano       │
│ Database: PostgreSQL, Redis        │
└─────────────────────────────────────┘
```

### Elixir/Erlang 스택 (분산)
```
┌─────────────────────────────────────┐
│ Framework: Phoenix, Cowboy         │
│ Concurrency: Actor model (OTP)     │
│ Distribution: Built-in clustering  │
│ Fault tolerance: Supervision trees │
└─────────────────────────────────────┘
```

## 실제 사례

### Riot Games (리그 오브 레전드)
- 결정론적 시뮬레이션
- 30 Hz tick rate
- 지역 데이터 센터
- 마이크로서비스 아키텍처

### Blizzard (월드 오브 워크래프트)
- Zone 기반 월드 서버
- Realm sharding
- 페이징 기술
- Cross-realm 기술

### Epic Games (포트나이트)
- 30 Hz tick rate (20 Hz에서 증가)
- 관심 영역 관리를 위한 Replication graph
- 전용 서버 모델
- 클라우드 기반 확장 (AWS)

### Valve (CS:GO)
- 64-128 Hz tick rate
- Source 엔진 전용 서버
- 지연 보상
- 클라이언트 측 예측

## 학습 경로

1. **기초** (1-2주차)
   - 기본 네트워킹 (TCP/UDP)
   - 스레딩 기초
   - Event loop

2. **중급** (3-6주차)
   - 비동기 I/O (IOCP, epoll)
   - Lock-free 프로그래밍
   - Game loop 설계
   - 상태 복제

3. **고급** (7-12주차)
   - 분산 시스템
   - Sharding 전략
   - 성능 최적화
   - 내결함성

4. **전문가** (4-6개월차)
   - 커스텀 프로토콜
   - 대규모 아키텍처
   - 고급 물리 시뮬레이션
   - 안티 치트 시스템

## 참고 자료

- [Gaffer on Games](https://gafferongames.com/) - 네트워킹과 게임 물리
- [Game Programming Patterns](https://gameprogrammingpatterns.com/) - 디자인 패턴
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) - 네트워크 기초
- [1500 Archers on a 28.8](https://www.gamedeveloper.com/programming/1500-archers-on-a-28-8-examining-network-code-in-age-of-empires) - 에이지 오브 엠파이어 네트워킹
- [I Shot You First](https://technology.riotgames.com/) - Riot Games 엔지니어링 블로그

## 다음 단계

- [실시간 게임 서버](01-realtime-game-server.md)에서 FPS/MOBA/MMO 아키텍처 살펴보기
- [비실시간 서버](02-non-realtime-server.md)에서 웹 기반 게임 알아보기
- [MMO 아키텍처](03-mmo-architecture.md)에서 대규모 멀티플레이어 탐구하기
- [Game Loop 스레딩](04-game-loop-threading.md)에서 핵심 엔진 설계 학습하기
- [네트워킹 스레딩](05-networking-threading.md)에서 고성능 I/O 마스터하기
